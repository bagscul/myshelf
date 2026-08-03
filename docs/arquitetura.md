# Arquitetura: Plataforma de Estante Pessoal (Rails)

Documento de referência pra levar pro Claude Code e começar a implementação. Cobre schema de banco, organização de código, plano de sincronização com APIs externas e os pontos que ainda precisam de uma decisão sua.

---

## 1. Schema de banco de dados

A decisão mais importante aqui é separar dois mundos que têm lógicas bem diferentes: os "itens colecionáveis" (jogo, livro, álbum, anime, filme, série, instrumento, item de compra, item de programação) e o módulo de viagem, que tem uma estrutura de avaliação própria com múltiplos critérios e múltiplas fontes. Tentar forçar viagem dentro do modelo genérico ia deixar o JSONB bagunçado e a query de "nota média por critério" muito mais difícil de escrever. Então ficam dois núcleos separados.

### 1.1 Núcleo genérico: `collectible_items`

```
collectible_items
  id                  bigint (PK)
  category            string, not null   # game, book, album, anime, movie, tv_show,
                                          # instrument_riff, purchase_item, programming_problem,
                                          # programming_paper, programming_oss_project,
                                          # programming_kaggle_competition
  subcategory         string             # ex: "dorama_kr", "dorama_jp" dentro de tv_show,
                                          # "vinyl", "instrument_gear", "manga_edition",
                                          # "pc_part", "peripheral" dentro de purchase_item
  title               string, not null
  external_source     string             # "steam", "jikan", "tmdb", nil se manual
  external_id         string             # appid da Steam, mal_id do Jikan, id do TMDB
  external_rating     decimal(4,2)       # nota bruta como veio da fonte (0-100 Metacritic, 0-10 MAL etc)
  external_rating_scale string           # "0-100", "0-10" — guarda a escala original pra normalizar depois
  user_rating         decimal(3,2)       # sua nota pessoal, opcional, sempre 0-5
  status              string, not null   # want, in_progress, completed, dropped, on_hold
  cover_url           string
  metadata            jsonb, default {}
  synced_at           datetime
  created_at / updated_at
```

Índices: GIN em `metadata` (pra poder filtrar por campos internos tipo autor ou ano sem precisar de coluna nova), índice composto em `(category, status)` que é o filtro mais comum da grid, e índice único em `(external_source, external_id)` quando não for nulo, pra evitar duplicar item na sincronização.

O `metadata` carrega o que muda de categoria pra categoria. Alguns exemplos de shape que ele assume:

- Livro: `{ "author": "Dostoiévski", "year": 1866, "publisher": "..." }`
- Álbum: `{ "artist": "...", "genre": "Post-Rock", "decade": "2010s" }`
- Instrumento: `{ "instrument": "bass", "artist": "...", "song": "...", "riff_recorded": false }`
- Programação: `{ "platform": "Codeforces", "difficulty": "1800", "topic": "dp", "solved": true }`

Essa flexibilidade é o motivo de usar JSONB em vez de criar uma tabela por hobby. Você ganha em não ter que fazer dez migrations toda vez que quiser adicionar um hobby novo, mas perde um pouco em validação estrita — isso eu resolveria a nível de model, com um schema de validação por categoria (pode usar o gem `dry-schema` ou simplesmente um método de validação customizado que troca de regra conforme o `category`).

### 1.2 Listas e a relação muitos-para-muitos

```
lists
  id            bigint (PK)
  name          string, not null
  description   text
  position      integer
  created_at / updated_at

list_items
  id                    bigint (PK)
  list_id               bigint (FK -> lists)
  collectible_item_id    bigint (FK -> collectible_items)
  position              integer
  created_at / updated_at
```

Índice único em `(list_id, collectible_item_id)` pra não duplicar o mesmo item na mesma lista. Um jogo em "quero jogar" e em "RPGs favoritos" vira só duas linhas em `list_items` apontando pro mesmo `collectible_item`.

### 1.3 Módulo de viagem

```
destinations
  id            bigint (PK)
  name          string, not null      # cidade
  region        string                # ex: "Região Metropolitana de Natal"
  country       string, default "Brasil"
  notes         text
  created_at / updated_at

establishments
  id                      bigint (PK)
  destination_id          bigint (FK -> destinations)
  name                    string, not null
  kind                    string, not null   # restaurant, attraction, lodging, transport_hub
  google_place_id         string
  tripadvisor_location_id string
  address                 string
  raw_metadata            jsonb, default {}  # payload bruto da API, pra não perder nada
  synced_at               datetime
  created_at / updated_at

establishment_ratings
  id                   bigint (PK)
  establishment_id      bigint (FK -> establishments)
  source               string, not null   # "google_places" ou "tripadvisor"
  place_score          decimal(2,1)       # 0 a 5, meio ponto permitido
  transport_score      decimal(2,1)
  lodging_score        decimal(2,1)
  activities_score     decimal(2,1)
  safety_score         decimal(2,1)
  fetched_at           datetime
  created_at / updated_at
```

Aqui optei por deixar os cinco critérios como colunas fixas em vez de normalizar numa tabela `rating_criteria` à parte. Cinco critérios fixos (local, transporte, hospedagem, atividades, segurança) dificilmente vão mudar, e ter colunas fixas deixa o cálculo da média muito mais direto: soma as colunas, divide por quantas fontes preencheram aquele critério, sem precisar de group by. Se um dia você quiser adicionar um sexto critério, é uma migration simples de `add_column`.

A nota final que aparece na grid (aquela que substitui o "nota manual" dos outros hobbies) é calculada, não armazenada: uma view ou um método no model que faz a média entre as linhas de `establishment_ratings` daquele estabelecimento, ponderando Google Places como fonte principal se quiser dar mais peso a ela.

### 1.4 Auditoria

PaperTrail resolve isso sozinho, criando a tabela `versions` automaticamente. Basta declarar `has_paper_trail` nos models `CollectibleItem` e `Establishment` (e nos outros que fizerem sentido) que ele já guarda quem mudou, quando e o valor anterior, sem precisar desenhar tabela de log manualmente. Só lembre de configurar `PaperTrail.request.whodunnit` no `ApplicationController` pra capturar o usuário logado.

---

## 2. Estrutura de pastas

```
app/
  models/
    collectible_item.rb
    list.rb
    list_item.rb
    destination.rb
    establishment.rb
    establishment_rating.rb
    concerns/
      metadata_validatable.rb        # validação condicional por categoria

  services/
    external_apis/
      steam_client.rb
      jikan_client.rb
      tmdb_client.rb
      google_places_client.rb
      tripadvisor_client.rb
    sync/
      steam_sync_service.rb          # decide criar/atualizar collectible_item
      jikan_sync_service.rb
      tmdb_sync_service.rb
      travel_sync_service.rb         # cuida de destination + establishment + ratings juntos

  jobs/
    sync/
      steam_sync_job.rb
      jikan_sync_job.rb
      tmdb_sync_job.rb
      travel_sync_job.rb

  controllers/
    collectible_items_controller.rb
    lists_controller.rb
    destinations_controller.rb
    establishments_controller.rb

  views/
    collectible_items/
      _card.html.erb                 # partial da capa no grid, reusado em toda categoria
    shared/
      _tabs.html.erb                 # navegação por categoria

  javascript/
    controllers/
      hover_card_controller.js       # Stimulus: mostra nota/metadado no hover
      filter_tabs_controller.js
```

A separação entre `external_apis/` e `sync/` é proposital: o client só sabe falar com a API (autenticação, request, parse da resposta), e o serviço de sync é quem decide a regra de negócio (criar, atualizar, ignorar). Assim, se um dia o Jikan mudar de endpoint ou você trocar de provedor de dados de anime, só mexe no client, a lógica de sync continua igual.

---

## 3. Plano de sincronização

O fluxo é parecido pros cinco jobs, então descrevo o padrão geral e depois as particularidades de cada um.

**Padrão geral de cada `*SyncJob`:**

1. Job roda no Sidekiq, geralmente agendado (sidekiq-cron ou similar) pra rodar uma vez por dia, mais um botão manual de "sincronizar agora" no ActiveAdmin pra quando você quiser forçar.
2. O job chama o serviço de sync correspondente, passando o `external_id` (ou a lista completa, no caso da wishlist da Steam).
3. O serviço busca no client, recebe o payload.
4. Verifica se já existe `collectible_item` com aquele `(external_source, external_id)`. Se existir, atualiza `external_rating`, `metadata` e `synced_at`. Se não existir, cria o registro com `status: "want"` como padrão.
5. Erros de rede ou de API fora do ar são capturados e reenfileirados com backoff exponencial (o Sidekiq já tem isso pronto via `sidekiq-retry` nativo, não precisa reinventar). Depois de um número de tentativas (sugiro 5), loga o erro e segue pro próximo item sem travar o job inteiro — um item com erro não pode impedir a sincronização dos outros noventa.
6. Log estruturado de cada rodada: quantos itens criados, quantos atualizados, quantos falharam. Isso ajuda muito quando você for debugar sem precisar ficar lendo request por request.

**Particularidades:**

- **Steam**: a wishlist inteira vem de um único endpoint, então o job busca a lista completa e itera, criando o que for novo e ignorando o que já existe sem mudança (checando um hash do payload pra não fazer update desnecessário).
- **Jikan**: por ser uma API não oficial baseada no MyAnimeList, tem rate limit mais agressivo. Vale colocar um `sleep` entre requests ou usar o `Faraday` com um middleware de retry que respeita o header de rate limit, pra não tomar bloqueio temporário.
- **TMDB**: filme e série usam o mesmo client, mudando só o endpoint (`/movie` vs `/tv`). Doramas ficam com `subcategory: "dorama_kr"` ou `"dorama_jp"` dentro da própria categoria `tv_show`, decidido a partir do `origin_country` que a TMDB retorna.
- **Google Places + TripAdvisor**: esse é o único fluxo que sincroniza duas fontes pro mesmo registro. O job busca em ambas, grava uma linha em `establishment_ratings` pra cada fonte que respondeu, e a nota final é sempre calculada em cima das linhas existentes. Se o TripAdvisor estiver fora do ar num dia, o Google Places sozinho já sustenta a nota, só com peso total nele.

---

## 4. Pontos em aberto

**Autenticação.** Pra um sistema de uso pessoal, eu não traria o Devise inteiro. Ele resolve muito mais problema do que você tem: recuperação de senha por email, confirmação de conta, múltiplos usuários. O gerador de autenticação nativo do Rails 8 (`bin/rails generate authentication`) já entrega sessão, `has_secure_password` e um model de usuário simples, com muito menos gem pra manter atualizada. Só vale considerar Devise se em algum momento você quiser abrir o sistema pra mais gente.

**Normalização de nota entre categorias.** Metacritic vem em 0-100, MAL em 0-10, TMDB em 0-10, sua nota pessoal em 0-5. Pra grid ficar visualmente consistente (mesma quantidade de estrelas em qualquer categoria), sugiro guardar `external_rating` na escala original (pra não perder precisão) e calcular um `normalized_rating` só na hora de exibir, dividindo por 20 quando a escala for 0-100 ou por 2 quando for 0-10. Isso pode virar um método simples no model, não precisa de coluna a mais no banco.

**Onde entra programação no modelo genérico.** As quatro sublistas (Codeforces/LeetCode, papers, projetos open source, competições Kaggle) são bem diferentes entre si, mas todas têm em comum um status de "resolvido/lido/estudado ou não", então cabem dentro de `collectible_items` com `category: "programming_problem"` etc e o `metadata` carregando o que for específico de cada sublista (dificuldade e tópico pro problema, autores e ano pro paper, linguagem principal pro projeto open source). Não vi motivo forte pra separar em tabela própria, mas se no futuro você quiser adicionar campos estruturados demais (tipo um checklist de conceitos estudados por paper), pode valer a pena revisitar isso.

**Peso entre Google Places e TripAdvisor.** Hoje o documento trata Google Places como fonte principal, mas não ficou definido se isso é média simples ou ponderada. Minha sugestão é começar com peso 70/30 (Places/TripAdvisor) e deixar esse número como constante configurável, já que é bem provável que você queira ajustar depois de ver os dados reais.

**ActiveAdmin x múltiplas categorias.** Como todos os hobbies compartilham a mesma tabela, o ActiveAdmin vai listar tudo junto por padrão. Vale configurar um `scope` por categoria dentro do `ActiveAdmin.register CollectibleItem` pra você não precisar filtrar manualmente toda vez que for cadastrar algo pelo painel.

---

Isso fecha o desenho geral. A parte que eu deixaria pra decidir só quando já estiver com o Claude Code na mão é o formato exato das views Hotwire (como o Turbo Frame de cada aba vai se comportar), porque aí faz mais sentido já ir testando no navegador em vez de especificar tudo às cegas aqui.
