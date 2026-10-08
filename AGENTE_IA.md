# Guia para Agente IA — Chatwoot MoveisBack

Documento de referência único para realizar manutenção, atualizações e personalizações neste projeto.

---

## 1. Visão Geral da Arquitetura

Este projeto constrói uma imagem Docker customizada sobre o fork `fazer-ai/chatwoot`.

**Pipeline:**
```
GitHub push → GitHub Actions → Dockerfile.full (build) → ghcr.io/fabricio-back/chatwoot-gc:latest → Coolify (deploy)
```

**Build em dois estágios (`Dockerfile.full`):**
- **Stage 1** (`node:24-alpine`): clona `fazer-ai/chatwoot` na tag fixada, copia os arquivos Vue/JS customizados, executa `pnpm build` (Vite + SDK)
- **Stage 2** (`ghcr.io/fazer-ai/chatwoot:<tag>`): copia os assets Vite compilados + todos os arquivos Ruby customizados

**Versão atual fixada:** `v4.18.0-fazer-ai.126`
A tag aparece em dois lugares no `Dockerfile.full`:
1. `git clone --branch v4.18.0-fazer-ai.126` (Stage 1)
2. `FROM ghcr.io/fazer-ai/chatwoot:v4.18.0-fazer-ai.126` (Stage 2)

> **IMPORTANTE:** A tag usada no `FROM` do Stage 2 DEVE existir no GHCR (`ghcr.io/fazer-ai/chatwoot`), não apenas no GitHub. Se a imagem Docker para aquela tag não existir no registry, o build falha. Em caso de dúvida, use `latest` no Stage 2 e fixe apenas o clone no Stage 1.

---

## 2. Mapa de Arquivos Customizados

Todos os arquivos customizados estão na pasta `custom/`. O `Dockerfile.full` copia cada um para o destino correto dentro da imagem.

### Frontend (Vue/JS) — compilados no Stage 1

| Arquivo local | Destino no build | O que faz |
|---|---|---|
| `custom/Sidebar.vue` | `app/javascript/dashboard/components-next/sidebar/Sidebar.vue` | Remove itens do menu para agentes (Conexões, Projetos, etc.) |
| `custom/InternalChat.vue` | `app/javascript/dashboard/components-next/sidebar/InternalChat.vue` | Widget de chat interno entre agentes no sidebar |
| `custom/LoginIndex.vue` | `app/javascript/v3/views/login/Index.vue` | "Powered by verticegrowth.com" na tela de login |
| `custom/theme-colors.js` | `theme/colors.js` | Paleta verde (`#23c93e`) no lugar do azul padrão do Chatwoot |
| `custom/conversations_getters.js` | `app/javascript/dashboard/store/modules/conversations/getters.js` | Adiciona getter `getChatGroupTypeFilter`; `getMineChats` inclui participantes |
| `custom/conversations_helpers.js` | `app/javascript/dashboard/store/modules/conversations/helpers.js` | `applyRoleFilter` restringe agentes a assignee+participante; adiciona sort `priority_desc_created_at_asc` |
| `custom/KanbanIndex.vue` | `app/javascript/dashboard/routes/dashboard/kanban/Index.vue` | Kanban de etiquetas completo (drag-and-drop, filtros, histórico, notas) |

> `ChatList.vue` **não é sobrescrito** desde v4.12 — a versão upstream já tem groupType, virtual scroll (virtua/vue) e permissões corretas.

### Patches inline no Stage 1 (via `sed` no Dockerfile.full)

| Arquivo patchado | O que muda |
|---|---|
| `dashboard/components-next/message/Message.vue` | `isRight` usa sender ID apenas para `Channel::Api` (Chat Interno); demais canais usam lógica original |
| Qualquer `*.js/*.ts/*.vue` com `kanban.moveisback.com.br` | URL neutralizada para `http://localhost:65535` |

### Backend (Ruby) — copiados no Stage 2

| Arquivo local | Destino na imagem | O que faz |
|---|---|---|
| `custom/permission_filter_service.rb` | `app/services/conversations/permission_filter_service.rb` | Agentes veem conversas onde são assignee OU participante |
| `custom/conversation_finder.rb` | `app/finders/conversation_finder.rb` | Backend do filtro "Minhas" inclui participantes; adiciona `filter_by_group_type`, `perform_meta_only`, sort `priority_desc_created_at_asc` |
| `custom/_conversation.json.jbuilder` | `app/views/api/v1/conversations/partials/_conversation.json.jbuilder` | Adiciona campo `is_participant` na resposta da API |
| `custom/saleshub_brand.rb` | `config/initializers/saleshub_brand.rb` | Aplica logos no banco ao iniciar |
| `custom/notification_listener.rb` | `app/listeners/notification_listener.rb` | Apenas admins recebem notificação de conversa criada |
| `custom/search_service.rb` | `app/services/search_service.rb` | Aplica `PermissionFilterService` na busca (agentes não veem conversas de outros) |
| `custom/internal_chat_inbox.rb` | `config/initializers/internal_chat_inbox.rb` | Cria automaticamente o inbox "Chat Interno" (Channel::Api) ao bootar |
| `custom/remove_kanban_script.rb` | `config/initializers/remove_kanban_script.rb` | Remove dashboard script do kanban externo antigo do banco |
| `custom/account_dashboard_patch.rb` | `config/initializers/account_dashboard_patch.rb` | Toggle "Agentes veem todas as conversas" + campo "Cor primária do tema" no Super Admin |
| `custom/account_theme_initializer.rb` | `config/initializers/account_theme_initializer.rb` | Rota `GET /account_theme/:account_id` — retorna cor do tema por conta |
| `custom/enterprise_unlock.rb` | (referenciado em outro initializer) | Desbloqueia features enterprise sem licença |
| `custom/vueapp.html.erb` | `app/views/layouts/vueapp.html.erb` | Injeta CSS do tema de cores dinâmico + fix anti-duplicata Kanban |

### Assets estáticos

| Pasta local | Destino | O que contém |
|---|---|---|
| `brand-assets/` | `/app/public/brand-assets/` | `logo.png`, `logo-dark.png`, `favicon.ico` |

---

## 3. Como Atualizar a Versão do Chatwoot

### Passo a passo

1. **Verificar se a nova tag existe** no GitHub: `https://github.com/fazer-ai/chatwoot/tags`
2. **Verificar se a imagem Docker existe** no GHCR: `https://github.com/fazer-ai/chatwoot/pkgs/container/chatwoot` — procurar a tag. Se não existir, usar `latest` no Stage 2.
3. **Analisar o diff** entre a versão atual e a nova nos arquivos que sobrescrevemos:
   ```
   https://github.com/fazer-ai/chatwoot/compare/v4.12.0-fazer-ai.54...vNOVA_TAG
   ```
   Arquivos críticos para verificar:
   - `app/javascript/dashboard/store/modules/conversations/getters.js`
   - `app/javascript/dashboard/store/modules/conversations/helpers.js`
   - `app/finders/conversation_finder.rb`
   - `app/javascript/dashboard/components-next/sidebar/Sidebar.vue`
   - `app/javascript/dashboard/components-next/message/Message.vue` (patch sed)
   - `app/javascript/dashboard/routes/dashboard/kanban/Index.vue`

4. **Mesclar mudanças upstream** nos arquivos customizados:
   - Adicionar novos métodos/getters/sort-options que o upstream introduziu
   - Preservar nossa lógica customizada (participantes, filtros de role, etc.)

5. **Atualizar a tag em dois lugares no `Dockerfile.full`:**
   ```dockerfile
   # Stage 1
   git clone --branch vNOVA_TAG ...
   # Stage 2
   FROM ghcr.io/fazer-ai/chatwoot:vNOVA_TAG
   ```

6. **Commit e push** → GitHub Actions builda automaticamente → aguardar ~2 min → Redeploy no Coolify.

### Erros comuns ao atualizar

| Sintoma | Causa provável | Solução |
|---|---|---|
| Build falha em ~1min com `/build/public/vite not found` | Vite não terminou — alguma importação quebrada no Vue customizado | Verificar se algum componente sobrescrito importa lib removida upstream |
| Build falha em ~1min com `manifest not found` | Tag do Stage 2 não existe no GHCR | Usar `ghcr.io/fazer-ai/chatwoot:latest` no Stage 2 |
| `NoMethodError` no Rails após deploy | Upstream adicionou método obrigatório que nosso arquivo Ruby customizado não tem | Comparar diff e adicionar o método ausente |
| Getter `undefined` no frontend | Upstream adicionou getter que nosso `conversations_getters.js` não tem | Adicionar o getter ausente no arquivo customizado |
| Conversas não carregam para agentes | `permission_filter_service.rb` ou `conversation_finder.rb` desatualizado | Comparar com upstream e mesclar |

---

## 4. Como Fazer Novas Personalizações

### Personalizações Frontend (Vue/JS)

1. Copiar o arquivo original do repositório upstream (na tag atual)
2. Aplicar as modificações
3. Salvar em `custom/`
4. Adicionar linha `COPY` no `Dockerfile.full` Stage 1 (antes do `pnpm install`)
5. Commit e push

**Regra:** Nunca editar arquivos que o upstream muda frequentemente sem analisar o diff a cada atualização.

### Personalizações Backend (Ruby)

1. Copiar o arquivo original da imagem base ou do repo upstream
2. Aplicar modificações
3. Salvar em `custom/`
4. Adicionar linha `COPY` no `Dockerfile.full` Stage 2 (após o `FROM`)
5. Commit e push

### Patches inline (sem sobrescrever arquivo inteiro)

Para mudanças pequenas em arquivos que não vale manter uma cópia completa, usar `sed` no Stage 1:
```dockerfile
RUN sed -i 's|texto_original|texto_novo|g' /build/caminho/do/arquivo.vue
```
Sempre adicionar `|| echo "WARN: patch falhou"` ao final para o build não travar silenciosamente.

### Inicializadores Rails

Para lógica que precisa rodar no boot do Rails (criar registros, patches de monkey-patch, etc.), criar um arquivo em `custom/` e copiar para `config/initializers/`. Use `rescue => e; Rails.logger.error` para não derrubar o boot em caso de erro.

---

## 5. Funcionalidades Customizadas — Detalhes

### Controle de Acesso de Agentes
- **Backend:** `permission_filter_service.rb` + `conversation_finder.rb`
- Agentes veem conversas onde são assignee OU participante
- Para liberar tudo para agentes: comentar a chamada de `PermissionFilterService` no `conversation_finder.rb`
- Toggle "Agentes veem todas as conversas" disponível no Super Admin por conta (`account_dashboard_patch.rb`)

### Tema de Cores por Conta
- Cor padrão: `#23c93e` (verde) — definida em `custom/theme-colors.js`
- Cada conta pode ter cor própria via Super Admin → campo "Cor primária do tema"
- O frontend busca a cor via `GET /account_theme/:account_id` (`account_theme_initializer.rb`)
- O CSS dinâmico é injetado em `vueapp.html.erb`

### Chat Interno entre Agentes
- Inbox `Channel::Api` criado automaticamente pelo initializer `internal_chat_inbox.rb`
- Widget no sidebar: `custom/InternalChat.vue`
- Mensagens alinhadas por sender ID (patch no `Message.vue`) apenas para `Channel::Api`

### Kanban de Etiquetas (`custom/KanbanIndex.vue`)
- Substitui o paywall do kanban_view do fazer-ai
- Drag-and-drop entre colunas (chama `POST /conversations/:id/labels`)
- Notas por card (salvas no contato via `POST /contacts/:id/notes`)
- Histórico de etiquetas: lê mensagens de atividade (`message_type: 2`) da conversa
  - Formato pt-BR: `"{user_name} adicionou {label}"` / `"{user_name} removeu {label}"`
  - Exibido no popup do card E disponível via ícone de relógio no card
- Filtro por assignee e busca por nome/ID
- Gerenciador de colunas (ocultar/reordenar, persistido em localStorage)

### Suporte a Grupos WhatsApp
- Ativado via variável de ambiente `BAILEYS_WHATSAPP_GROUPS_ENABLED=true`
- Frontend filtra por `group_type` via `getChatGroupTypeFilter` getter (adicionado em `conversations_getters.js`)
- Backend filtra via `filter_by_group_type` em `conversation_finder.rb`

---

## 6. Infraestrutura (docker-compose.coolify.yml)

Serviços em produção:
- `rails` — aplicação principal (porta 3000)
- `sidekiq` — workers (`SIDEKIQ_CONCURRENCY=25`)
- `postgres` — `ghcr.io/fazer-ai/postgres-16-pgvector:latest`
- `redis` — `redis:alpine`
- `baileys-api` — `ghcr.io/fazer-ai/baileys-api:latest` (WhatsApp)

Variáveis de ambiente obrigatórias no Coolify:
- `FRONTEND_URL` — URL pública do Chatwoot
- `MAILER_SENDER_EMAIL`, `RESEND_API_KEY` — e-mail
- `BRAND_ASSETS_URL` — ZIP com logo/favicon (opcional)
- `BAILEYS_PROVIDER_DEFAULT_CLIENT_NAME` — nome do cliente WhatsApp

---

## 7. Fluxo de Deploy

```
1. Editar arquivo em custom/ (ou Dockerfile.full)
2. git add . && git commit -m "..." && git push origin master
3. GitHub Actions inicia build automaticamente
4. Aguardar ~2 minutos (build com cache) ou ~20 minutos (build frio)
5. Se build OK: acessar Coolify → serviço → Force Redeploy
6. Coolify puxa a nova imagem e reinicia os containers
```

**Verificar build:** `https://github.com/fabricio-back/chatwoot-gc/actions`

---

## 8. Referências Rápidas

- Repo de customizações: `https://github.com/fabricio-back/chatwoot-gc`
- Repo upstream: `https://github.com/fazer-ai/chatwoot`
- Tags disponíveis: `https://github.com/fazer-ai/chatwoot/tags`
- Imagens GHCR do upstream: `https://github.com/fazer-ai/chatwoot/pkgs/container/chatwoot`
- Registry da imagem customizada: `ghcr.io/fabricio-back/chatwoot-gc:latest`
