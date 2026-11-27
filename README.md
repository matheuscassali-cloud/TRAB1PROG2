# Programação II e Redes Aplicada a Jogos 2026/2

Repositório com os projetos desenvolvidos para a Atividade Acadêmica **Programação II e Redes Aplicada a Jogos**, parte do curso de **Pós-Graduação em Design de Jogos (PUCRS)** — Prof. Miguel Xavier e Prof.ª Rossana B. Queiroz.

## Equipe e principal função
- Matheus Cassali — *função a preencher*
- Angelo Almeida — *função a preencher*

## Estrutura do Repositório

Os trabalhos são evoluções do mesmo jogo, por isso todos vivem no **mesmo projeto Unreal**, na raiz deste repositório.

| Projeto | Engine / Linguagem | Descrição breve |
| ------- | ------------------ | --------------- |
| `T1` — Kart Multiplayer | Unreal Engine 5.8 (Blueprints) | Corrida de kart em rede (Listen Server): lobby, pista de 2 voltas, contagem de voltas e tela de vencedor |
| Refinamento — Animação, Navegação e Interface | Unreal Engine 5.8 (Blueprints) | NPC fiscal com AnimBP (State Machine, Blend Space, Anim Notify) patrulhando via NavMesh, contagem regressiva de largada e HUD orientado a eventos |
| `T2` — Multiplayer Competitivo em Ambiente Compartilhado | Unreal Engine 5.8 (Blueprints) | Até 4 jogadores na mesma pista: ciclo de partida controlado pelo servidor, voltas e pontuação no PlayerState, cronômetro no GameState, pickups de boost disputados e validação de input no servidor |

> 💡 Observações
> * Este repositório deverá permanecer **público** até o final da disciplina para fins de avaliação.
> * A **reflexão sobre consistência e segurança** do T2 está neste README, na seção [Reflexão sobre Consistência e Segurança](#reflexão-sobre-consistência-e-segurança).
> * O memorial descritivo técnico do refinamento (animação/navegação/interface) está em [`docs/Memorial_T2.pdf`](docs/Memorial_T2.pdf).

---

# Kart Multiplayer — T2: Multiplayer Competitivo em Ambiente Compartilhado

---

## Descrição do Projeto

Corrida de kart para **2 a 4 jogadores em rede** (Listen Server). O jogo abre num **menu inicial**, onde o host escolhe quantos jogadores a partida terá (2 a 4) e cria a sala; os outros entram pelo IP. Os jogadores se encontram no lobby (`MainMap`), dirigem até o botão de largada e são levados à pista (`NewMap`). Quando o número mínimo de jogadores está conectado, o servidor inicia a contagem **“3, 2, 1, JÁ!”**; quem completar **2 voltas** primeiro vence, e o tempo final aparece para todos.

Todos os karts disputam os mesmos **pickups de boost** espalhados pela pista: cada pickup só pode ser pego por **um** jogador, dá um impulso, some para todos e reaparece depois de alguns segundos. Cada boost coletado conta na pontuação individual do jogador.

Toda decisão que importa (quem pegou o boost, quem completou a volta, quem venceu, quando a corrida começa e termina) é tomada **pelo servidor**; os clientes só enviam input e recebem o estado replicado.

---

## Ciclo de vida da partida

O estado da partida fica no GameState (`EstadoPartida`, enum `E_EstadoPartida`, RepNotify) e só é alterado pelo GameMode, que existe apenas no servidor:

```
Aguardando ──(jogadores conectados ≥ JogadoresNecessarios)──► Contagem ──(3, 2, 1)──► EmCorrida ──(alguém completa 2 voltas)──► Finalizada
```

| Estado | O que acontece | Quem decide |
| ------ | -------------- | ----------- |
| **Aguardando** | Karts parados; HUD mostra “Aguardando jogadores…” | `GM_Kart.OnPostLogin` conta os jogadores no `PlayerArray` e compara com `JogadoresNecessarios` (escolhido no menu e guardado no `BP_GI_Kart`) |
| **Contagem** | 3-2-1 na tela (`Contagem`, RepNotify); input de aceleração ignorado | Timer `PassoContagem` no `GM_Kart` |
| **EmCorrida** | `CorridaComecou = true`; servidor grava `InicioCorrida` com o relógio do servidor; cronômetro corre | `GM_Kart.PassoContagem` quando a contagem chega a 0 |
| **Finalizada** | Servidor grava `TempoFinal`, o NPC comemora e a tela de vencedor é mostrada a todos | `GM_Kart.EndRace_Event`, chamado pela linha de chegada |

**Máximo de jogadores:** 4 (`MaxPlayers=4` em `Config/DefaultGame.ini` e 4 PlayerStarts na pista). O 5º jogador é recusado pelo servidor.

**Entrada tardia:** quem conecta com a partida fora do estado *Aguardando* entra como **espectador**. O `GM_Kart` sobrescreve `HandleStartingNewPlayer` e só chama a implementação padrão (que cria o kart) se a partida ainda estiver aguardando; o `OnPostLogin` também não dá número de jogador nem conta esse jogador para a largada.

---

## Estrutura do Projeto

| Arquivo | Descrição |
| ------- | --------- |
| `Content/Variant_Timetrial/Maps/MenuPrincipal`, `Content/W_MenuPrincipal`, `Content/GM_Menu` | Menu inicial: escolher 2–4 jogadores, **Criar partida** (abre o lobby com `?listen`), **Entrar** (`open <IP>`) e **Sair** |
| `Content/BP_GI_Kart` | GameInstance: guarda `JogadoresNecessarios` escolhido no menu (sobrevive à troca de mapa; só o valor do host é usado) |
| `Content/GM_Kart` | GameMode (só no servidor): controla o ciclo de vida, a contagem, a entrada tardia (`HandleStartingNewPlayer`), a escolha de PlayerStart e o fim da corrida |
| `Content/BP_GS_Kart` | GameState replicado: `EstadoPartida` (RepNotify), `Contagem` (RepNotify), `CorridaComecou`, `InicioCorrida`, `TempoFinal` |
| `Content/BP_PS_Kart` | PlayerState replicado: `Voltas` (RepNotify + dispatcher `OnVoltasMudou`), `BoostsColetados`, `NumeroJogador`; funções `AdicionarVolta` e `AdicionarBoost` só alteram com `Has Authority` |
| `Content/E_EstadoPartida` | Enum: Aguardando, Contagem, EmCorrida, Finalizada |
| `Content/BP_BoostPickup` | Recurso compartilhado: `Ativo` (RepNotify) esconde/mostra o pickup e liga/desliga a colisão; overlap tratado só no servidor |
| `Content/BP_FinishLine` | Linha de chegada: overlap com o kart conta a volta no servidor (verifica o sentido pela spline) |
| `Content/BP_TrackSpline` | Spline que descreve o traçado da pista |
| `Content/BP_BotaoLobby` | Gatilho do lobby que leva os jogadores à pista (`ServerTravel`) |
| `Content/VehicleTemplate/NewCarsBP/BP_KartPhysics` | Kart com física (Simulate Physics); recebe input por `Server_SetAcceleration` / `Server_SetSteer` (com Clamp no servidor) e mostra o fim de jogo por `Multicast_EndGame` |
| `Content/NPC/Blueprints/BP_NPC_Fiscal`, `BP_AIC_Fiscal` | NPC fiscal: patrulha via NavMesh e comemora no fim da corrida |
| `Content/NPC/Animations/ABP_NPC_Fiscal`, `BS1D_NPC_Locomocao` | Animation Blueprint e Blend Space do NPC |
| `Content/Variant_Timetrial/UI/W_HUD` | HUD local de cada jogador: contém `W_LapCount`, `W_Contagem` e `W_Cronometro` |
| `Content/Variant_Timetrial/UI/W_LapCount` | Voltas do jogador local: escuta `OnVoltasMudou` do próprio PlayerState |
| `Content/Variant_Timetrial/UI/W_Cronometro` | Estado da partida e cronômetro, calculado localmente a partir do relógio do servidor |
| `Content/Variant_Timetrial/UI/W_EndGameUI` | Tela de fim de jogo com o vencedor |
| `Content/Variant_Timetrial/Maps/MainMap`, `NewMap` | Lobby e pista de corrida |

---

## Informações Técnicas

- **Engine:** Unreal Engine 5.8
- **Linguagem:** Blueprints
- **Dependências:** Enhanced Input, Third Person (Manny); karts com física própria (o plugin Chaos Vehicles foi desativado)
- **Rede:** Listen Server, 2 a 4 jogadores (servidor autoritativo)
- **Plataforma-alvo:** Windows

---

## Checklist de Requisitos (T2 — Redes)

- [x] Menu inicial: criar partida (host escolhe 2–4 jogadores) ou entrar por IP
- [x] Todos os jogadores no mesmo nível, com máximo planejado (4)
- [x] Consumo/alteração de recurso compartilhado (pickups de boost, RepNotify)
- [x] Ciclo de vida controlado pelo servidor: aguardando → em andamento → fim com resultado
- [x] GameState com estado global e cronômetro
- [x] PlayerState com pontuação e informações individuais
- [x] Política de entrada tardia (espectador) e limite de jogadores
- [x] Validação de input no servidor (Clamp nos Server RPCs)
- [x] Reflexão sobre condições de corrida, consistência/latência e trapaças (abaixo)
- [ ] Build para o playtest de 27/11

---

## Como jogar

Controles: **W/S** acelerar e frear · **A/D** virar · **R** reposicionar o kart.

Abra **`Jogar.bat`** (ou `Trabalho1.exe`) na pasta da build. O jogo abre no menu inicial:

1. **Host:** escolha o número de jogadores com **−/+** (2 a 4) e clique em **Criar partida**. O jogo vira servidor (Listen Server) e abre o lobby.
2. **Demais jogadores:** digite o IP do host e clique em **Entrar**. Campo vazio = `127.0.0.1` (mesma máquina). Em outra máquina da rede, use o IP local do host (`ipconfig`) e permita o jogo no Firewall do Windows.

No lobby, dirija até o botão de largada para ir à pista. A contagem começa quando o número de jogadores escolhido pelo host estiver conectado. Passe pelos pickups para ganhar impulso e complete 2 voltas antes dos outros.

---

## Reflexão sobre Consistência e Segurança

### Quem é dono de cada informação

| Informação | Onde vive | Quem altera | Quem precisa saber | Quando é enviada |
| ---------- | --------- | ----------- | ------------------ | ---------------- |
| Estado da partida (`EstadoPartida`) | GameState | Servidor (`GM_Kart`) | Todos (HUD) | RepNotify, só quando muda de fase |
| Contagem 3-2-1 (`Contagem`) | GameState | Servidor (timer) | Todos (HUD) | RepNotify, 1×/s durante a contagem |
| `CorridaComecou` | GameState | Servidor | Servidor (aceita aceleração) e clientes | Replicated, uma vez |
| `InicioCorrida` / `TempoFinal` | GameState | Servidor | Todos (cronômetro) | Replicated, uma vez cada; o cliente calcula o tempo decorrido localmente |
| Voltas do jogador (`Voltas`) | PlayerState | Servidor (linha de chegada) | Todos; a HUD do dono reage | RepNotify, a cada volta |
| Boosts coletados | PlayerState | Servidor (pickup) | Todos (placar) | Replicated, a cada coleta |
| Número do jogador | PlayerState | Servidor (`OnPostLogin`) | Todos (tela de vencedor) | Replicated, ao conectar |
| Pickup ativo (`Ativo`) | BP_BoostPickup | Servidor | Todos (visual e colisão) | RepNotify, ao ser pego e ao reaparecer |
| Posição/velocidade do kart | Kart (física) | Servidor (simulação) | Todos | Replicação de movimento, contínua |
| Input de aceleração/direção | Kart | **Cliente** (dono) | Servidor | Server RPC, a cada mudança de input |
| HUD, câmera, efeitos | Cliente local | Cliente | Só o próprio jogador | Não replicado |

### 1. Condições de corrida (race conditions)

**Dois karts pegando o mesmo boost no mesmo milissegundo.** O overlap só é tratado no servidor (`Has Authority`), e o servidor processa os eventos de overlap **um de cada vez**, na mesma thread do jogo. O primeiro evento a ser processado testa `Ativo == true` e, como **primeira ação**, grava `Ativo = false`; só depois aplica o impulso e soma o boost. O segundo evento encontra `Ativo == false` e é descartado. Não existe estado em que os dois ganhem. A ordem entre os dois, porém, é a ordem em que a **física do servidor** detectou o contato, e não “quem apertou primeiro”: com latência diferente, o jogador com ping maior chega ao pickup “mais tarde” no servidor do que na própria tela.
*Tratamento:* manter a regra “o servidor decide” e tornar a perda explícita para o jogador, com um som ou efeito de “pickup já pego”. Uma melhoria possível é favorecer quem tem ping maior com uma tolerância (lag compensation), mas isso abre espaço para abuso; preferimos a regra simples.

**Dois karts cruzando a linha de chegada na última volta no mesmo frame.** Os dois overlaps chamam `EndRace_Event` em sequência no servidor. O evento começa com `Branch(NOT RaceFinished)` e grava `RaceFinished = true` antes de qualquer outra coisa, então só o primeiro processado vira vencedor. O critério de desempate é, de novo, a ordem de processamento do servidor.
*Tratamento proposto:* registrar o tempo exato de cruzamento de cada kart (interpolando a posição entre dois frames de física) e vencer quem tiver o menor tempo, em vez de “quem foi processado primeiro”.

**Vários jogadores conectando ao mesmo tempo.** O `OnPostLogin` dispara uma vez por jogador; a contagem é protegida por um `DoOnce`, então ela só começa uma vez mesmo que dois logins cheguem juntos. O `NumeroJogador` é calculado como o tamanho atual do `PlayerArray`: se alguém sair e outro entrar, dois jogadores podem acabar com o mesmo número.
*Tratamento proposto:* usar um contador que só cresce no GameMode (ou o `PlayerId` da engine) em vez do tamanho do array.

**Boost reaparecendo com um kart parado em cima.** Quando o pickup reativa, o kart que já está dentro da esfera não gera um novo *Begin Overlap*, então não ganha o boost até sair e entrar de novo. É um comportamento consistente em todas as máquinas, só não é o mais intuitivo.

### 2. Consistência de dados e latência

**Sim, os clientes veem um estado diferente do servidor, sempre com atraso.** O servidor é a única fonte da verdade; cada cliente vê uma versão um pouco antiga dela (cerca de meio RTT para chegar a informação). Os pontos onde isso aparece no nosso jogo:

- **Movimento do próprio kart.** O input vai ao servidor por RPC, a física roda no servidor e a posição volta por replicação. Sem predição no cliente, o jogador sente um atraso de ~1 RTT entre apertar a tecla e o kart reagir; com perda de pacote, o kart dos outros pode “pular”. *Tratamento:* predição local do movimento com correção pelo servidor (o que o `CharacterMovementComponent` faz para personagens), ou suavização (interpolação) dos karts remotos.
- **Pickup “fantasma”.** O cliente continua vendo um pickup que outro jogador já pegou até o RepNotify de `Ativo` chegar. Ele pode passar por cima e não ganhar nada. Como o boost é decidido só pelo servidor, isso nunca gera um boost indevido, só uma frustração visual. *Tratamento:* esconder o pickup localmente assim que o kart encosta (feedback otimista) e mostrá-lo de novo se o servidor não confirmar.
- **Boost atrasado.** O impulso é aplicado no servidor; o cliente só vê o efeito quando a nova velocidade é replicada.
- **Cronômetro.** O servidor não envia o tempo a cada frame: envia só `InicioCorrida` uma vez, e cada cliente calcula `GetServerWorldTimeSeconds − InicioCorrida`. O relógio do servidor no cliente é uma estimativa sincronizada periodicamente, então pode haver diferença de alguns centésimos entre telas. O `TempoFinal` é calculado no servidor e é igual para todos.
- **Voltas e vencedor.** São decididos no servidor e chegam por RepNotify/Multicast. Pode haver um instante em que um cliente ainda mostra “1/2” enquanto o servidor já registrou a volta, mas o resultado final é sempre o do servidor.
- **Host saindo.** É um Listen Server: se o host fecha o jogo, a partida termina para todos (não há migração de host). Se um cliente cai no meio da corrida, o PlayerState e o kart dele somem e a corrida continua; se todos os outros saírem, a partida fica sem vencedor. *Tratamento proposto:* servidor dedicado para o playtest e, no GameMode, encerrar a corrida quando restar só um jogador.

**Teste com rede degradada (Network Emulation no PIE):**
*a preencher com o que foi observado no teste com perfil “Bad”/“Average” (atraso de resposta do kart, pulos dos karts remotos, pickup fantasma, diferença do cronômetro) e prints.*

### 3. Autoridade e trapaças (cheating)

**O que é confiado ao cliente:** apenas o **input** (aceleração e direção, valores entre -1 e 1) e o que é puramente local (HUD, câmera). Tudo o que define o resultado (posição física do kart, voltas, boosts, vencedor, estado e tempo da partida) é calculado no servidor.

| Tentativa de trapaça | O que impede no nosso jogo |
| -------------------- | -------------------------- |
| Mandar aceleração 50× maior pelo RPC | `Server_SetAcceleration` e `Server_SetSteer` aplicam `Clamp(-1, 1)` antes de usar o valor |
| Acelerar antes da largada | O servidor ignora a aceleração (`Select` com `CorridaComecou`) enquanto a corrida não começou |
| Dar voltas a mais / alterar `Voltas` | `Voltas` só muda no servidor (`AdicionarVolta` com `Has Authority`), a partir do overlap físico com a linha; o sentido é verificado pela spline e cruzar de ré desconta a volta |
| Pegar boosts à distância ou repetidos | O overlap é avaliado no servidor com a posição autoritativa do kart; o pickup trava (`Ativo = false`) até reaparecer |
| Teleportar o kart | A posição vem da simulação física do servidor; o cliente não envia posição |
| Declarar-se vencedor | O vencedor é decidido no `GM_Kart`, que só existe no servidor |

**O que ainda é vulnerável:**
- **Atalhos na pista.** A linha de chegada só verifica o sentido, não o caminho. Um jogador pode cortar caminho fora da pista, se a geometria permitir. *Tratamento:* checkpoints obrigatórios em ordem, validados no servidor.
- **Reposicionar o kart (R).** É um Server RPC que o cliente pode chamar quantas vezes quiser; dependendo de onde ele reposiciona, pode ser usado para ganhar vantagem. *Tratamento:* cooldown no servidor e reposicionar sempre no último checkpoint validado.
- **Spam de RPCs.** Um cliente modificado pode enviar milhares de `Server_SetAcceleration` por segundo, gastando banda e CPU do host. *Tratamento:* limitar a taxa de RPCs por jogador no servidor.
- **O host é um jogador.** Num Listen Server, o host roda o servidor e tem autoridade total; um host malicioso pode alterar qualquer coisa. *Tratamento:* servidor dedicado.
- **Automação de input (bots).** O servidor não tem como distinguir um humano de um script que gera o input ideal; isso exigiria detecção por padrões de comportamento, fora do escopo do trabalho.

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
- Textura de madeira dos karts — *Planks012*, [ambientCG](https://ambientcg.com) (CC0)
- Modelo da tampa de garrafa das rodas — *fonte a preencher*
- Referências de interface: *Mario Kart* e *Crash Team Racing*

---

## Comentários Finais (opcional)

- O número de jogadores para a largada é escolhido pelo host no menu inicial (2 a 4) e guardado em `JogadoresNecessarios` no `BP_GI_Kart`; o máximo é 4.
- O memorial descritivo técnico do refinamento (animação, navegação e interface) está em [`docs/Memorial_T2.pdf`](docs/Memorial_T2.pdf).
