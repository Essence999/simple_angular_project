# Projeto de Angular

Testando a estrutura de projeto do Angular e os comandos elementares.

import anyio
import aiohttp
from enum import Enum
from litestar import Litestar, Controller, post, get, State
from litestar.background_tasks import BackgroundTask
from pydantic import BaseModel

# ... (Enums e Models permanecem iguais)

class DocumentDownloader:
    # ... (__init__, get_pending_ids, _worker, _download_document permanecem iguais)

    async def _execute_pool(self, pending_ids: list[str], token: str) -> None:
        headers = {"Authorization": f"Bearer {token}"}
        try:
            async with aiohttp.ClientSession(headers=headers) as session:
                send_stream, receive_stream = anyio.create_memory_object_stream(math.inf)
                
                async with anyio.create_task_group() as tg:
                    for _ in range(self.max_concurrency):
                        tg.start_soon(self._worker, receive_stream.clone(), session)
                    
                    async with send_stream:
                        for doc_id in pending_ids:
                            await send_stream.send(doc_id)
            
            self.state = DownloadState.COMPLETED

        # Captura erros empacotados do AnyIO TaskGroup e erros de I/O
        except *aiohttp.ClientError as e:
            self.state = DownloadState.ERROR
            self.error_msg = f"Erro de rede: {e.exceptions[0]}"
        except *OSError as e:
            self.state = DownloadState.ERROR
            self.error_msg = f"Erro de disco: {e.exceptions[0]}"
        except ExceptionGroup as eg:
            self.state = DownloadState.ERROR
            self.error_msg = f"Falha de processamento: {eg.exceptions}"

    async def start_download(self, all_ids: list[str], token: str) -> None:
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

        await self._execute_pool(pending_ids, token)


class DownloadController(Controller):
    path = "/download"

    @post()
    async def trigger_download(self, state: State, token: str) -> dict:
        ids_from_db = ["101", "102", "103", "104"] 
        
        # Uso de BackgroundTask nativa do Litestar. Sem atrito de threads.
        task = BackgroundTask(state.downloader.start_download, ids_from_db, token)
        
        return {"status": "accepted", "background": task}

# ... (app e get_status permanecem iguais)
