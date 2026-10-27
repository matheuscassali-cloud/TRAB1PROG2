# Programação II e Redes Aplicada a Jogos 2026/2

Repositório com os projetos desenvolvidos para a Atividade Acadêmica **Programação II e Redes Aplicada a Jogos**, parte do curso de **Pós-Graduação em Design de Jogos (PUCRS)** — Prof.ª Rossana B. Queiroz.

## Equipe e principal função
- Matheus Cassali — *função a preencher*
- Angelo Almeida — *função a preencher*

## Estrutura do Repositório

O T2 é a evolução do T1 (o enunciado permite refinar o projeto anterior), por isso os dois trabalhos vivem no **mesmo projeto Unreal**, na raiz deste repositório.

| Projeto | Engine / Linguagem | Descrição breve |
| ------- | ------------------ | --------------- |
| `T1` — Kart Multiplayer | Unreal Engine 5.8 (Blueprints) | Corrida de kart para 2 jogadores em rede (Listen Server): lobby, pista de 2 voltas, contagem de voltas por jogador e tela de vencedor |
| `T2` — Refinamento: Animação, Navegação e Interface | Unreal Engine 5.8 (Blueprints) | NPC fiscal com AnimBP (State Machine, Blend Space, Anim Notify) patrulhando via NavMesh, contagem regressiva de largada e HUD orientado a eventos |

> 💡 Observações
> * Este repositório deverá permanecer **público** até o final da disciplina para fins de avaliação.
> * O memorial descritivo técnico do T2 está em [`docs/Memorial_T2.pdf`](docs/Memorial_T2.pdf).

---

# Kart Multiplayer — T2: Animação, Navegação e Interface

---

## Descrição do Projeto

Jogo de corrida de kart para **2 jogadores em rede**. Os jogadores se encontram no lobby (`MainMap`), dirigem até o botão de largada e são levados à pista (`NewMap`). Após a contagem **“3, 2, 1, JÁ!”**, quem completar **2 voltas** primeiro vence. Na lateral da pista, um **NPC fiscal** patrulha usando a NavMesh, com animações de andar e correr e som de passos; quando a corrida termina, ele para e pula comemorando.

O T2 refina o T1 com foco em **Animação** (malha esquelética + Animation Blueprint), **Navegação** (NavMesh + AIController) e **Interface** (UMG orientada a eventos, sem Tick/Binding).

---

## Estrutura do Projeto

| Arquivo | Descrição |
| ------- | --------- |
| `Content/GM_Kart` | GameMode (só no servidor): inicia a contagem quando 2 jogadores entram, encerra a corrida e define o vencedor |
| `Content/BP_GS_Kart` | GameState replicado: `Contagem` (RepNotify) e `CorridaComecou`; avisa a UI pelo dispatcher `OnContagemMudou` |
| `Content/BP_FinishLine` | Linha de chegada: overlap com o kart conta a volta no servidor (verifica o sentido pela spline) |
| `Content/BP_TrackSpline` | Spline que descreve o traçado da pista |
| `Content/BP_BotaoLobby` | Gatilho do lobby que leva os jogadores à pista (`ServerTravel`) |
| `Content/VehicleTemplate/NewCarsBP/BP_KartPhysics` | Kart com física (Simulate Physics), RPCs de input, `CurrentLap` (RepNotify) e `Multicast_EndGame` |
| `Content/NPC/Blueprints/BP_NPC_Fiscal` | NPC fiscal (Character): patrulha e comemoração com pulo |
| `Content/NPC/Blueprints/BP_AIC_Fiscal` | AIController: patrulha entre Target Points com `AI MoveTo` |
| `Content/NPC/Animations/ABP_NPC_Fiscal` | Animation Blueprint: State Machine Locomoção → Pulo → Queda → Aterrissagem |
| `Content/NPC/Animations/BS1D_NPC_Locomocao` | Blend Space 1D: Idle / Andar / Correr pela velocidade |
| `Content/NPC/Audio/SW_Passo`, `ATT_Passo` | Som de passo disparado pelo Anim Notify `Passo`, com atenuação por distância |
| `Content/Variant_Timetrial/UI/W_HUD` | HUD: contém `W_LapCount` (voltas) e `W_Contagem` (largada) |
| `Content/Variant_Timetrial/UI/W_EndGameUI` | Tela de fim de jogo com o vencedor |
| `Content/Variant_Timetrial/Maps/MainMap`, `NewMap` | Lobby e pista de corrida |

---

## Informações Técnicas

- **Engine:** Unreal Engine 5.8
- **Linguagem:** Blueprints
- **Dependências:** Enhanced Input, Chaos Vehicles Plugin (template de veículos), Third Person (Manny)
- **Rede:** Listen Server, 2 jogadores
- **Plataforma-alvo:** Windows

---

## Checklist de Requisitos (T2)

- [x] Malha esquelética controlada por Animation Blueprint
- [x] State Machine com locomoção por Blend Space 1D e estados de salto/queda
- [x] Anim Notify disparando evento em tempo de execução (som de passo)
- [x] NavMeshBoundsVolume no nível
- [x] Agente autônomo (NPC) navegando entre pontos de patrulha
- [x] Câmera com SpringArm: TargetArmLength, Camera Lag e colisão de câmera (Probe Size)
- [x] HUD em UMG com informações em tempo real (voltas, contagem)
- [x] Atualização da interface orientada a eventos (RepNotify + Event Dispatchers), sem Tick/Binding
- [x] Feedback de estado de jogo (contagem de início e tela de vitória)
- [x] Perfis de colisão explícitos (canal `Kart`, presets `Kart` e `LinhaChegada`) e física de corpo rígido

---

## Como jogar

Controles: **W/S** acelerar e frear · **A/D** virar · **R** reposicionar o kart.

Na pasta da build há dois atalhos:
1. **`Host.bat`** — abre o jogo como servidor (Listen Server) no lobby.
2. **`Join.bat`** — conecta ao host. Na mesma máquina, aperte Enter (usa `127.0.0.1`); em outra máquina da rede, digite o IP do host.

No lobby, dirija até o botão de largada para ir à pista. A corrida começa quando os 2 jogadores estão conectados.

---

## Link para a Build

🔗 *link do itch.io a preencher* (senha: `PUCRSJOGOS2026`)

---

## Vídeo de Apresentação (se aplicável)

🎬 *a preencher*

---

## Referências e/ou créditos

- Unreal Engine — templates **Vehicle** e **Third Person** (malha Manny e animações), Epic Games
- Som de passo — *fonte a preencher*
- Referências de interface: *Mario Kart* e *Crash Team Racing*

---

## Comentários Finais (opcional)

- A corrida exige 2 jogadores conectados para a contagem começar (`JogadoresNecessarios = 2` no `GM_Kart`).
- O memorial descritivo técnico completo está em [`docs/Memorial_T2.pdf`](docs/Memorial_T2.pdf).
