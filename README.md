# Lili — Voice Chat

A Discord-style web and Windows desktop client for messaging, voice, video and screen sharing, built on Supabase and LiveKit/TURN. There's no demo-data fallback: everything runs against a real backend.

**[Open the web app](https://lilivoicechat-five.vercel.app)** · Current version: **0.2.3** · [Release notes](docs/CHANGELOG.md) · [🇧🇷 Português](#-português)

## Features

**Navigation and servers**
- Two contexts that never mix, like Discord: Home (friends, direct messages and message requests) and one server at a time. The URL follows the context (`#/channels/@me`, `#/channels/<server>/<channel>`), so refresh, pasted links and back/forward land in the same place.
- Server bar with ordering and folders; server profile with name, icon (crop, zoom, reposition) and description; servers without an icon use a monogram.
- Channels created through a modal (text, voice or category, privacy decided upfront); channels created inside a category inherit its permissions.
- Discord-style channel editor: overview (name, topic, slow mode, voice limit) and per-role or per-member permissions with deny/inherit/allow, category resync and deletion.
- Invite links (`/invite/<code>`) that unfurl in Discord, WhatsApp or Slack with the server name, member count, landing channel and icon; revoked or expired invites fall back to a generic card.

**Messages**
- Replies, reactions, pins, edits, deletion, Markdown, spoilers (text and attachments) and user/role/`@everyone`/`@here` mentions.
- Attachments up to 100 MB with inline image and video preview and a fullscreen viewer; files live 24 hours on the server, after which the recipient can ask the sender to resend. Paste images straight into the composer.
- Voice messages with recording in the composer and a custom player.
- GIF picker (Giphy, with the API key kept server-side in an Edge Function) and emoji picker; the chosen GIF is downloaded and sent as an attachment instead of a link.
- Group DMs with member selection, name, private icon and access management.

**Calls and screen sharing**
- Ringing calls: the caller sees "Calling…" and can cancel; the callee gets a modal over any screen to answer with audio, video or decline. Answered, declined, cancelled and missed are persisted and synced, and call history only shows calls you took part in.
- LiveKit SFU for audio, video and screen, with configurable UDP/TLS TURN; noise suppression on the microphone.
- Screen sharing tuned for motion, with quality settings that actually reach the encoder.
- On the desktop app, sharing an application sends **only that application's audio** (per-process capture on Windows 10+), and system-audio capture excludes Lili's own process tree, so there's no echo.
- Independent volume per live stream, kept while focusing, going fullscreen or restarting the share.
- A diagnostics panel with real resolution and frame rate, encoder (GPU or CPU), bitrate target, estimated bandwidth, keyframes and oversized frames.
- Voice moderation: mute, deafen, move and disconnect.

**Accounts and notifications**
- E-mail code verification on sign-up (delivered through Brevo), password recovery by e-mail link with a dedicated screen; changing the password ends other sessions. Revocable sessions and a real DB/Storage quota dashboard.
- Accounts without login for 90 days become tombstones: access and identity are destroyed, conversations are kept, and orphaned servers pass to the oldest admin.
- Web Push/VAPID with `ALL`, `MENTIONS` and `NONE` preferences, plus native/web foreground notifications with GLOBAL/SERVER/CHANNEL scopes and temporary mute.
- Custom sound identity synthesized with Web Audio (looping ringtone, join and leave sounds triggered only by real presence changes).
- In-app release notes after each update.

## Security model

- Access control is Supabase Auth plus Row Level Security: reading requires channel membership and sending requires write permission, enforced by the database on every query. Transport is encrypted (HTTPS for the API, DTLS-SRTP for media).
- **Messages are not end-to-end encrypted.** E2EE with OpenMLS was removed in version 0.1.2 as a scope decision: messages and attachments are stored readable by the backend, and LiveKit media is not end-to-end encrypted either. Whoever operates the instance must treat the database and the SFU as trusted surfaces and cover admin access, retention and data handling in the privacy policy.
- Electron runs with sandbox, `contextIsolation`, validated IPC and DPAPI via `safeStorage`. Camera, microphone and screen sharing require explicit OS/browser permission.

## Stack

React · TypeScript · Vite · Zustand · TanStack Query · Dexie · Supabase (Auth, PostgreSQL, RLS, transactional RPCs, Realtime, private Storage, Edge Functions) · LiveKit · Electron (NSIS, tray, `electron-updater`) · Vercel · Vitest · pgTAP · Playwright

## Running the local stack

Requirements: Node.js 22+, Docker Desktop with virtualization enabled, PowerShell 7 or Windows PowerShell.

Copy `.env.example` to `.env.local`, fill in the publishable key from `npx supabase status -o env`, and create `supabase/functions/.env.local` with the local LiveKit secrets. Never put a service-role key in the frontend.

```powershell
npm ci
npm run local:up
```

This prepares Docker Desktop, starts Supabase and LiveKit/TURN, applies migrations and keeps Vite plus the Edge Functions running. Open `http://127.0.0.1:5173/` and keep the terminal open. The LiveKit script detects the Windows LAN IPv4 and writes `infra/livekit/livekit.local.yaml`; if needed, set it manually:

```powershell
powershell -File scripts/configure-livekit-local.ps1 -NodeIp 192.168.1.20
```

To validate two accounts and local media end to end:

```powershell
npm run test:local
```

It creates and removes temporary accounts and tests the shared workspace, chat and a real call in two browser contexts, LiveKit/TURN, desktop share audio, the database/RLS, unit tests, typecheck and build. For hardware acceptance, also follow the [manual checklist](docs/MORNING_TEST_CHECKLIST.md).

## Tests

```powershell
npm run typecheck
npm test                     # Vitest: app, Edge Functions and API
npm run test:db              # pgTAP + schema lint
npm run test:workspace
npm run test:chat            # Playwright
npm run test:livekit         # two synthetic participants, requires a TURN relay candidate
npm run test:update-installed
npm run build
```

## Desktop

```powershell
npm run desktop:dev     # app over the local development stack
npm run desktop:prod    # app over production infrastructure
npm run desktop:dist    # installer pointed at production
npm run desktop:smoke   # smoke test of the desktop bundle
```

Since 0.1.3 the installed app loads the **published site** (with an offline page as fallback) instead of a bundled copy, so web and desktop always run the same version. The installer only packages against production: an executable talking to `127.0.0.1` wouldn't work on anyone else's machine. Artifacts land in `release/` with `SHA256SUMS.txt`. The release workflow publishes the installer, blockmap, `latest.yml` and the version's notes (cut from `docs/CHANGELOG.md`) to GitHub Releases; the local installer stays `NotSigned` until a code-signing certificate is provided.

## Production

The full path is in [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md), in the order it must be followed, because each step produces a value the next one needs:

```
Supabase  →  LiveKit/TURN  →  Vercel (web)  →  GitHub Releases (desktop)
```

```powershell
npm run deploy:supabase -- --project-ref YOUR_PROJECT_REF   # schema, secrets and Edge Functions
npm run build:web                                           # the same build Vercel runs
```

`build:web` validates the public config before compiling: it rejects `127.0.0.1` URLs in production, a service-role key pasted as the publishable key, and any secret with a `VITE_` prefix (everything with that prefix ships in plain text to the browser). `vercel.json` handles SPA rewrites, caching and security headers; the strict CSP is built from the environment's real origins in `vite.config.ts`.

---

## 🇧🇷 Português

# Lili — Voice Chat

Cliente web e desktop Windows no estilo Discord para mensagens, voz, vídeo e compartilhamento de tela, sobre Supabase e LiveKit/TURN. Não há fallback com dados de demonstração: tudo roda contra um backend real.

**[Abrir o app web](https://lilivoicechat-five.vercel.app)** · Versão atual: **0.2.3** · [Notas de versão](docs/CHANGELOG.md)

### Recursos

**Navegação e servidores**
- Dois contextos que nunca se misturam, como no Discord: a Home (amigos, conversas diretas e solicitações de mensagem) e um servidor por vez. O endereço acompanha o contexto (`#/channels/@me`, `#/channels/<servidor>/<canal>`), então atualizar, colar um link ou usar voltar/avançar levam ao mesmo lugar.
- Barra de servidores com ordenação e pastas; perfil do servidor com nome, ícone (recorte, zoom e reposicionamento) e descrição; servidor sem ícone usa o monograma do nome.
- Canais criados por modal (texto, voz ou categoria, com privacidade decidida antes); canal criado dentro de categoria herda as permissões dela.
- Editor de canal como no Discord: visão geral (nome, tópico, modo lento, limite de voz) e permissões por cargo ou membro com negar/herdar/permitir, ressincronização com a categoria e exclusão.
- Links de convite (`/invite/<codigo>`) que viram cartão no Discord, WhatsApp ou Slack com nome do servidor, número de membros, canal de entrada e ícone; convite revogado ou vencido volta ao cartão genérico.

**Mensagens**
- Respostas, reações, pins, edição, exclusão, Markdown, spoiler (em texto e anexo) e menções de usuário, cargo, `@everyone` e `@here`.
- Anexos de até 100 MB com prévia de imagem e vídeo no chat e visualizador em tela cheia; os arquivos ficam 24 h no servidor e depois quem perdeu pede o reenvio. Dá para colar imagem direto no compositor.
- Mensagens de voz, com gravação no compositor e player próprio.
- Seletor de GIFs (Giphy, com a chave guardada no servidor numa Edge Function) e de emoji; o GIF escolhido é baixado e enviado como anexo, não como link.
- Conversas em grupo com escolha de membros, nome, ícone privado e gestão de acesso.

**Chamadas e compartilhamento de tela**
- Chamadas com toque: quem liga vê "Chamando…" e pode cancelar; quem recebe vê um modal acima de qualquer tela para atender com áudio, com vídeo ou recusar. Atendida, recusada, cancelada e não atendida ficam salvas e sincronizadas, e o histórico só mostra chamadas de que você participou.
- LiveKit SFU para áudio, vídeo e tela, com TURN UDP/TLS configurável e supressão de ruído no microfone.
- Compartilhamento de tela ajustado para conteúdo em movimento, com qualidade que de fato chega ao codificador.
- No app desktop, compartilhar um aplicativo envia **só o som daquele aplicativo** (captura por processo no Windows 10+), e a captura do som do sistema exclui os processos do próprio Lili, então não há eco.
- Volume próprio para cada live, mantido ao focar, abrir em tela cheia ou reiniciar o compartilhamento.
- Painel de diagnóstico com resolução e quadros reais, codificador (GPU ou CPU), alvo de taxa, banda estimada, keyframes e quadros enormes.
- Moderação de voz: silenciar, ensurdecer, mover e desconectar.

**Contas e notificações**
- Verificação por código de e-mail no cadastro (entregue pelo Brevo) e recuperação de senha por link, com tela própria; trocar a senha encerra as outras sessões. Sessões revogáveis e painel real de cota de banco e Storage.
- Conta sem login por 90 dias vira lápide: acesso e identidade são destruídos, a conversa fica, e servidor órfão passa para o administrador mais antigo.
- Web Push/VAPID com preferências `ALL`, `MENTIONS` e `NONE`, e notificação nativa/web em primeiro plano com escopos GLOBAL/SERVER/CHANNEL e silêncio temporário.
- Identidade sonora própria sintetizada com Web Audio (toque em laço, sons de entrada e saída disparados só por mudança real de presença).
- Notas da versão dentro do app depois de cada atualização.

### Modelo de segurança

- O controle de acesso é Supabase Auth mais Row Level Security: ler exige participar do canal e enviar exige permissão de escrita, aplicado pelo banco a cada consulta. O transporte é cifrado (HTTPS na API e DTLS-SRTP na mídia).
- **As mensagens não têm criptografia de ponta a ponta.** O E2EE com OpenMLS foi removido na versão 0.1.2 por decisão de escopo: mensagens e anexos ficam legíveis para o backend, e a mídia do LiveKit também não é cifrada ponta a ponta. Quem opera a instância precisa tratar banco e SFU como superfícies de confiança e cobrir acesso administrativo, retenção e tratamento de dados na política de privacidade.
- O Electron roda com sandbox, `contextIsolation`, IPC validado e DPAPI via `safeStorage`. Câmera, microfone e compartilhamento exigem autorização explícita do sistema ou do navegador.

### Stack

React, TypeScript, Vite, Zustand, TanStack Query, Dexie, Supabase (Auth, PostgreSQL, RLS, RPCs transacionais, Realtime, Storage privado e Edge Functions), LiveKit, Electron (NSIS, tray e `electron-updater`), Vercel, Vitest, pgTAP e Playwright.

### Rodar a stack local

Requisitos: Node.js 22+, Docker Desktop com virtualização habilitada e PowerShell 7 ou Windows PowerShell.

Copie `.env.example` para `.env.local`, preencha a publishable key de `npx supabase status -o env` e crie `supabase/functions/.env.local` com os segredos locais do LiveKit. Nunca coloque service-role key no frontend.

```powershell
npm ci
npm run local:up
```

O comando prepara o Docker Desktop, sobe Supabase e LiveKit/TURN, aplica as migrations e mantém Vite e Edge Functions rodando. Acesse `http://127.0.0.1:5173/` e deixe o terminal aberto. O script do LiveKit detecta o IPv4 da LAN e gera `infra/livekit/livekit.local.yaml`; se precisar, informe manualmente com `scripts/configure-livekit-local.ps1 -NodeIp 192.168.1.20`.

Para validar duas contas e a mídia local de ponta a ponta, rode `npm run test:local`. Ele cria e remove contas temporárias e testa o workspace compartilhado, chat e chamada real em dois contextos de navegador, LiveKit/TURN, áudio do compartilhamento no desktop, banco/RLS, testes unitários, typecheck e build. Para o aceite com hardware, siga também o [checklist manual](docs/MORNING_TEST_CHECKLIST.md).

### Testes

`npm run typecheck`, `npm test` (Vitest do app, Edge Functions e API), `npm run test:db` (pgTAP e lint do schema), `npm run test:workspace`, `npm run test:chat` (Playwright), `npm run test:livekit` (dois participantes sintéticos, exige candidato TURN relay), `npm run test:update-installed` e `npm run build`.

### Desktop

`npm run desktop:dev` (app sobre a stack local), `npm run desktop:prod` (app sobre a produção), `npm run desktop:dist` (instalador apontado para produção) e `npm run desktop:smoke` (teste do bundle desktop).

Desde a 0.1.3, o app instalado carrega o **site publicado** (com uma página offline de reserva), em vez de uma cópia embutida, então web e desktop rodam sempre a mesma versão. O instalador só empacota contra produção: um executável que fala com `127.0.0.1` não funcionaria na máquina de mais ninguém. Os artefatos ficam em `release/` com `SHA256SUMS.txt`. O workflow de release publica instalador, blockmap, `latest.yml` e as notas da versão (recortadas de `docs/CHANGELOG.md`) no GitHub Releases; o instalador local fica `NotSigned` até existir um certificado de code signing.

### Produção

O caminho completo está em [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md), na ordem em que precisa ser seguido: Supabase → LiveKit/TURN → Vercel (web) → GitHub Releases (desktop).

```powershell
npm run deploy:supabase -- --project-ref YOUR_PROJECT_REF   # schema, segredos e Edge Functions
npm run build:web                                           # o mesmo build que a Vercel roda
```

O `build:web` valida a configuração pública antes de compilar: recusa URL de `127.0.0.1` em produção, uma service-role key colada no lugar da publishable key e qualquer segredo com prefixo `VITE_` (tudo com esse prefixo vai em texto puro para o navegador). O `vercel.json` cuida da reescrita de SPA, do cache e dos cabeçalhos de segurança; a CSP restrita é montada no `vite.config.ts` a partir das origens reais do ambiente.
