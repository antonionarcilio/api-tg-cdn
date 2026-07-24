# Referência de rotas

Toda rota roda atrás do middleware `requireToken` (`src/server.js`). "Privada"
abaixo significa que exige o header `Authorization: Bearer SEU_TOKEN`.
"Pública (URL assinada)" significa que não precisa de header — mas só é
acessível com um `sig`/`exp` válidos, escopados àquele recurso específico e
com expiração de 1h (nunca é "sem autenticação nenhuma"). Veja
[`../README.md`](../README.md#autenticação) para a explicação completa do
esquema de auth.

## `GET /routes`

- **Propósito**: lista todas as rotas registradas no servidor (introspecção de
  `router.stack` em tempo de execução — nunca fica desatualizada).
- **Acesso**: Privada.
- **Query params**: nenhum.
- **Resposta**: `[{ "method": "GET", "path": "/videos" }, ...]`

## `GET /videos`

- **Propósito**: percorre todos os seus chats/canais (`getDialogs`) e lista os
  vídeos encontrados em cada um, já com a URL de streaming pronta (assinada).
- **Acesso**: Privada.
- **Query params**: `limit` (opcional, default `10`) — máximo de vídeos por
  chat.
- **Resposta**: array de `{ chat_id, chat_title, message_id, file_name, size,
  mime_type, date, url }`.

## `GET /channels`

- **Propósito**: lista os canais/supergrupos de que você faz parte.
- **Acesso**: Privada.
- **Query params**: nenhum.
- **Resposta**: `[{ "chat_id": "...", "chat_title": "..." }, ...]`

## `GET /channels/:chatId/videos`

- **Propósito**: lista os vídeos de um canal específico.
- **Acesso**: Privada.
- **Query params**: `limit` (opcional, default `20`).
- **Resposta**: `{ chat_id, chat_title, data: [{ message_id, file_name, size,
  mime_type, date, url }] }` — `chat_id`/`chat_title` do canal ficam fora do
  array `data`.

## `GET /list/:chatId`

- **Propósito**: lista vídeos de qualquer chat (não só canais) por `chatId` —
  útil pra descobrir o `messageId` de um vídeo em "Saved Messages" (`chatId
  = me`) ou em qualquer conversa.
- **Acesso**: Privada.
- **Query params**: `limit` (opcional, default `20`).
- **Resposta**: array de `{ message_id, file_name, size, mime_type, date }`
  (sem `url` — essa rota é só pra descoberta, sem o wrapper de canal).

## `GET /video/:chatId/:messageId`

- **Propósito**: rota de streaming de fato — serve os bytes do vídeo com
  suporte a `Range` (206 Partial Content), buscando sob demanda do Telegram
  via `iterDownload`, sem baixar o arquivo inteiro em disco.
- **Acesso**: Híbrida — aceita **Privada** (header `Authorization`) **ou
  Pública (URL assinada)** via `?exp=...&sig=...`. É a única rota que aceita
  o segundo modo, porque precisa ser abrível direto por VLC/`<video src>`/
  navegador, que não anexam headers customizados numa navegação simples.
- **Query params**: `exp` (timestamp Unix de expiração) e `sig` (HMAC-SHA256
  de `chatId:messageId:exp`, ver `src/signedUrl.js`) — gerados
  automaticamente pelas rotas `/videos` e `/channels/:chatId/videos`; não
  monte esse par manualmente.
- **Resposta**: stream binário do vídeo (`200` ou `206`), não JSON.
