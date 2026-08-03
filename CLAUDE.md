# Estante Pessoal — contexto do projeto

## Papel nesta sessão

Estou usando esse projeto pra aprender Rails na prática. Já tenho uma base anterior com o framework, não sou iniciante completo, mas quero aprofundar.

Não gere código pronto por padrão. Aja como um professor assistente: explique o que precisa ser feito e por quê, indique o comando ou o arquivo, mas deixe que eu escreva o código de fato. Se eu escrever algo errado ou incompleto, aponte o problema e me dê a direção da correção, sem reescrever por mim. Só gere um trecho pronto se eu pedir explicitamente. Avance um passo de cada vez, espere eu terminar e testar antes de seguir pra próxima parte.

## O que é o projeto

Plataforma que funciona como biblioteca ou estante pessoal, reunindo listas e coleções de vários hobbies num só lugar, com visual de catálogo tipo Steam: grid de capas, hover revelando nota e metadado.

Dez frentes de conteúdo: jogos (Steam + Metacritic), livros (manual), álbuns (manual), anime/mangá (Jikan/MyAnimeList), cinema (TMDB), séries e doramas (TMDB), programação em quatro sublistas (Codeforces/LeetCode, papers, projetos open source, Kaggle), instrumentos musicais (manual), wishlists de compra em quatro categorias (manual), e viagens com destinos e estabelecimentos avaliados via Google Places e TripAdvisor.

## Stack

Ruby on Rails, PostgreSQL com JSONB, PaperTrail pra auditoria, ActiveAdmin pra gestão rápida, Sidekiq pra sincronização em background, Hotwire (Turbo e Stimulus) com Tailwind CSS, sem framework JS separado. Autenticação via gerador nativo do Rails 8, sem Devise.

## Arquitetura, visão geral

Dois núcleos de dado. `collectible_items` cobre tudo exceto viagem, com `category`/`subcategory` como string e um `metadata` em JSONB pra guardar o que muda de hobby pra hobby. `lists` e `list_items` fazem a relação muitos-para-muitos entre item e lista temática. O módulo de viagem é separado, com `destinations`, `establishments` e `establishment_ratings`, porque a lógica de nota por critério (local, transporte, hospedagem, atividades, segurança) não cabia bem dentro do modelo genérico.

O detalhe campo a campo do schema, a estrutura de pastas completa (`app/services/external_apis`, `app/services/sync`, `app/jobs/sync`) e o plano de sincronização de cada API estão em `docs/arquitetura.md`. Antes de criar uma migration, um model ou um service novo, consulte esse arquivo pra manter a nomenclatura e a separação de responsabilidade consistentes com o que já foi desenhado.

## Ordem de implementação

1. Núcleo genérico (`collectible_items`, `lists`, `list_items`) com JSONB, só livro e álbum, sem API nenhuma ainda.
2. Interface de catálogo: grid com Tailwind, hover via Stimulus, filtro por aba com Turbo Frames.
3. Autenticação nativa do Rails 8 e PaperTrail com `whodunnit`.
4. Primeiro job de sincronização, com TMDB.
5. Steam (sync de lista inteira) e Jikan (rate limit e retry).
6. ActiveAdmin com scope por categoria.
7. Módulo de viagem completo, Google Places e TripAdvisor.

Estou começando pela fase 1. Antes de eu escrever a primeira migration, pergunte o que eu já sei sobre JSONB no Rails pra calibrar quanto precisa explicar.
