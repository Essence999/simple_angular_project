# Projeto de Angular

Testando a estrutura de projeto do Angular e os comandos elementares.

 import anyio
import aiohttp
from enum import Enum
from litestar import Litestar, Controller, post, get, State
from pydantic import BaseModel

class DownloadState(str, Enum):
    IDLE = "IDLE"
    DOWNLOADING = "DOWNLOADING"
    COMPLETED = "COMPLETED"
    ERROR = "ERROR"

class DownloadStatusResponse(BaseModel):
    state: DownloadState
    total: int
    downloaded: int
    error: str | None = None

class DocumentDownloader:
    def __init__(self, max_concurrency: int = 10):
        self.max_concurrency = max_concurrency
        self.state = DownloadState.IDLE
        self.total_docs = 0
        self.downloaded_docs = 0
        self.error_msg: str | None = None
        self._lock = anyio.Lock()
        self.save_dir = anyio.Path("documentos")

    async def get_pending_ids(self, all_ids: list[str]) -> list[str]:
        await self.save_dir.mkdir(parents=True, exist_ok=True)
        existing_files = {f.name for f in await self.save_dir.iterdir() if f.is_file()}
        return [doc_id for doc_id in all_ids if f"{doc_id}.pdf" not in existing_files]

    async def _worker(self, receive_stream, session: aiohttp.ClientSession):
        async with receive_stream:
            async for doc_id in receive_stream:
                await self._download_document(doc_id, session)

    async def _download_document(self, doc_id: str, session: aiohttp.ClientSession):
        url = f"https://api.origem.com/docs/{doc_id}" # Exemplo
        tmp_path = self.save_dir / f"{doc_id}.pdf.tmp"
        final_path = self.save_dir / f"{doc_id}.pdf"

        async with session.get(url) as response:
            if response.status >= 400:
                raise RuntimeError(f"HTTP {response.status} ao baixar doc {doc_id}")
            
            async with await anyio.open_file(tmp_path, "wb") as f:
                async for chunk in response.content.iter_chunked(64 * 1024):
                    await f.write(chunk)
        
        await tmp_path.rename(final_path)
        self.downloaded_docs += 1

    async def start_download(self, all_ids: list[str], token: str):
        async with self._lock:
            if self.state == DownloadState.DOWNLOADING:
                return

            self.state = DownloadState.DOWNLOADING
            self.error_msg = None
            pending_ids = await self.get_pending_ids(all_ids)
            self.total_docs = len(pending_ids)
            self.downloaded_docs = 0

            if not pending_ids:
                self.state = DownloadState.COMPLETED
                return

        # Executa fora do lock para não bloquear requisições de status
        headers = {"Authorization": f"Bearer {token}"}
        try:
            async with aiohttp.ClientSession(headers=headers) as session:
                send_stream, receive_stream = anyio.create_memory_object_stream(math.inf)
                
                async with anyio.create_task_group() as tg:
                    for _ in range(self.max_concurrency):
                        tg.start_soon(self._worker, receive_stream.clone(), session)
                    
                    with send_stream:
                        for doc_id in pending_ids:
                            await send_stream.send(doc_id)
            
            self.state = DownloadState.COMPLETED

        except Exception as e:
            self.state = DownloadState.ERROR
            self.error_msg = str(e)


class DownloadController(Controller):
    path = "/download"

    @post()
    async def trigger_download(self, state: State, token: str) -> dict: # Token injetado via middleware/depends
        # Mock de IDs vindos do DB
        ids_from_db = ["101", "102", "103", "104"] 
        
        # Dispara em background via TaskGroup injetado pelo Litestar ou anyio.to_thread dependendo da config global.
        # Para concorrência estruturada pura, Litestar permite injetar AnyIO backend.
        anyio.from_thread.start_blocking_portal().start_task_soon(
            state.downloader.start_download, ids_from_db, token
        )
        return {"status": "accepted"}

    @get("/status")
    async def get_status(self, state: State) -> DownloadStatusResponse:
        dl = state.downloader
        return DownloadStatusResponse(
            state=dl.state,
            total=dl.total_docs,
            downloaded=dl.downloaded_docs,
            error=dl.error_msg
        )

app = Litestar(
    route_handlers=[DownloadController],
    state=State({"downloader": DocumentDownloader(max_concurrency=10)})
)
