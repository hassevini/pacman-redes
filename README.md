<a id="readme-top"></a>

[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![LinkedIn][linkedin-shield]][linkedin-url]

<br />
<div align="center">
  <h3 align="center">👾 Pacman-Redes</h3>

  <p align="center">
    Jogo estilo Pac-Man no modelo cliente-servidor sobre Ethernet — trabalho final da disciplina de Redes de Computadores 1 (CI1058) na UFPR.
    <br />
  </p>
</div>

---

<!-- SUMÁRIO -->
<details>
  <summary>Sumário</summary>
  <ol>
    <li><a href="#-sobre-o-projeto">Sobre o Projeto</a>
      <ul>
        <li><a href="#-construído-com">Construído com</a></li>
      </ul>
    </li>
    <li><a href="#-fundamentação--arquitetura">Fundamentação / Arquitetura</a></li>
    <li><a href="#-entidades--componentes">Entidades / Componentes</a></li>
    <li><a href="#-dinâmica-e-fluxo-de-execução">Dinâmica e Fluxo de Execução</a></li>
    <li><a href="#-módulos-do-protocolo">Módulos do Protocolo</a></li>
    <li><a href="#-estrutura-do-projeto">Estrutura do Projeto</a></li>
    <li>
      <a href="#-instalação">Instalação</a>
      <ul>
        <li><a href="#-pré-requisitos">Pré-requisitos</a></li>
        <li><a href="#-compilação">Compilação</a></li>
        <li><a href="#-execução">Execução</a></li>
        <li><a href="#-comandos-úteis">Comandos Úteis</a></li>
      </ul>
    </li>
    <li><a href="#-dificuldades-e-aprendizados">Dificuldades e Aprendizados</a></li>
    <li><a href="#-licença">Licença</a></li>
    <li><a href="#-contato">Contato</a></li>
    <li><a href="#-agradecimentos">Agradecimentos</a></li>
  </ol>
</details>

---

## 📖 Sobre o Projeto

![imagem exemplo do jogo em execução](assets/screenshots/running.png)

**Pacman-Redes** é um jogo estilo Pac-Man implementado no modelo **cliente-servidor**, desenvolvido em linguagem C para a disciplina **Redes de Computadores 1 (CI1058)** da **Universidade Federal do Paraná (UFPR)**.

A comunicação entre as duas máquinas é feita via **cabo Ethernet**, usando **raw sockets** e um protocolo inspirado no **Kermit** (simplificado e adaptado às necessidades do trabalho). O servidor concentra toda a lógica do jogo (posições, colisões, pastilhas, fantasmas), enquanto o cliente é responsável exclusivamente pela interface interativa com o jogador, renderizada via **ncurses**.

O jogo é **por turnos**: a cada jogada do usuário, o cliente envia um pacote de direção ao servidor, que processa o estado do mundo e devolve a visão atualizada — sem tempo contínuo correndo em segundo plano.

As **pastilhas** do mapa são arquivos reais (2 `.txt`, 2 `.jpg` e 2 `.mp4`) que, ao serem coletadas pelo Pac-Man, são transmitidas do servidor ao cliente pelo protocolo, exatamente como um download.

<p align="right">(<a href="#readme-top">voltar ao topo</a>)</p>

### 🛠 Construído com

* [![C][C-badge]][C-url]
* [![Linux][Linux-badge]][Linux-url]
* **ncurses** — interface de terminal no lado cliente
* **Raw Sockets (AF_PACKET)** — comunicação direta na camada de enlace
* **Protocolo Kermit (adaptado)** — controle de fluxo, sequenciamento e CRC-8

<p align="right">(<a href="#readme-top">voltar ao topo</a>)</p>

---

## ⏱ Arquitetura

A comunicação segue o modelo **simplex com confirmação (stop-and-wait)**: o remetente envia um pacote e aguarda ACK ou NACK antes de enviar o próximo. A detecção de erros é feita via **CRC-8** calculado sobre o payload do pacote.

O frame do protocolo Kermit adaptado possui a seguinte estrutura:

```
+----------------+----------+----------+----------+----------+----------+
| starter_marker | size (5) | seq (6)  | type (5) | data (n) | crc (8)  |
|    0x7E (8b)   |   bits   |   bits   |   bits   |  bytes   |   bits   |
+----------------+----------+----------+----------+----------+----------+
```

O campo `type` suporta até 32 tipos de mensagens distintos (ACK, NACK, VISUALIZACAO, DADOS, TXT, JPG, MP4, movimentos direcionais, VITORIA, DERROTA, etc.).

O fluxo geral do sistema:

```
+------------------------------------------------------------------+
|                     Fluxo do Sistema                             |
|                                                                  |
|  Cliente                          Servidor                       |
|  --------                         --------                       |
|  1. Conecta via raw socket    <-->  Aguarda conexão              |
|  2. Recebe mapa inicial (VISUALIZACAO)                           |
|  3. Renderiza interface ncurses                                  |
|  4. Usuário pressiona tecla → envia DIREITA/ESQUERDA/CIMA/BAIXO  |
|  5.                          ← Servidor processa update_world()  |
|  6. Recebe novo mapa / pastilha / VITORIA / DERROTA              |
|  7. Se pastilha: recebe arquivo em chunks (DADOS)                |
|  8. Repete 3-7 até fim de jogo                                   |
+------------------------------------------------------------------+
```

<p align="right">(<a href="#readme-top">voltar ao topo</a>)</p>

---

## 👥 Entidades

| Entidade | Atributos Principais | Descrição |
|-------------------|----------------------|-----------|
| **Pac-Man** (`pacman_t`) | posição (linha, coluna), raio de visão | Entidade controlada pelo jogador; o mapa enviado ao cliente é a submatriz dentro do raio de visão do Pac-Man. |
| **Fantasmas** (`ghost_t`) | posição (linha, coluna) × N instâncias | Entidades movem-se aleatoriamente a cada turno; colisão com Pac-Man encerra o jogo. |
| **Pastilhas** (`pellet_t`) | posição, id (1–6), tipo de arquivo | Objetos colecionáveis; ao serem coletados, o servidor transmite o arquivo correspondente ao cliente. |
| **Mapa** (`char map[N][N]`) | grade bidimensional de caracteres | Lido de um `.csv`, armazena o estado completo do mundo; o servidor constrói a submatriz de visão antes de enviar. |
| **Protocolo Kermit** (`kermit_t`) | marker, size, seq, type, data, crc | Frame de comunicação; serializado/deserializado a cada envio-recepção. |
| **Interface ncurses** (`interface.c`) | janelas, input de teclado | Lado cliente: renderiza o mapa recebido e captura o movimento do usuário. |

<p align="right">(<a href="#readme-top">voltar ao topo</a>)</p>

---

## 🔄 Dinâmica e Fluxo de Execução

| Evento | Ações Executadas |
|----------------|----------------------------------|
| `Início` | Servidor abre raw socket e aguarda; cliente abre raw socket e aguarda handshake. |
| `Mapa inicial` | Servidor envia o estado inicial do mapa (tipo `VISUALIZACAO`) em chunks; cliente renderiza via ncurses. |
| `Movimento do jogador` | Cliente captura tecla direcional e envia pacote (`DIREITA` / `ESQUERDA` / `CIMA` / `BAIXO`). |
| `Processamento do turno` | Servidor chama `update_world()`: move fantasmas aleatoriamente, move Pac-Man, checa colisões. |
| `Colisão com fantasma` | Servidor envia `DERROTA`; cliente exibe tela de game over e encerra. |
| `Coleta de pastilha` | Servidor detecta colisão com pastilha, inicia transmissão do arquivo (TXT/JPG/MP4) em chunks `DADOS`. |
| `Raio de visão` | Servidor monta submatriz centrada no Pac-Man (`RAIO`) e envia ao cliente a cada turno. |
| `Vitória` | Pac-Man coletou todas as 6 pastilhas; servidor envia `VITORIA`, cliente exibe tela de vitória. |
| `SAIR` | Qualquer lado pode enviar `SAIR` para encerrar a partida. |
| `ACK / NACK` | Receptor confirma pacote válido (CRC correto) com `ACK` ou solicita retransmissão com `NACK`. |

<p align="right">(<a href="#readme-top">voltar ao topo</a>)</p>

---

## 🧩 Módulos do Protocolo

O projeto é organizado em módulos com responsabilidades bem definidas:

* **`kermit` (`kermit.h` / `network/kermit.c`):** Implementação completa do protocolo — criação de frames, serialização/deserialização, cálculo de CRC-8, envio/recepção via raw socket com timeout e validação de marcador de início.
* **`game` (`game.h` / `core/game.c`):** Lógica central do servidor — leitura do mapa CSV, inicialização aleatória das entidades, detecção de colisões, construção da submatriz de visão e atualização do mundo por turno.
* **`pacman` (`pacman.h` / `core/pacman.c`):** Estrutura e movimentação do Pac-Man no mapa.
* **`ghosts` (`ghosts.h` / `core/ghosts.c`):** Estrutura e movimentação aleatória dos fantasmas.
* **`pellets` (`pellets.h` / `core/pellets.c`):** Estrutura e estado das pastilhas colecionáveis.
* **`interface` (`interface.h` / `core/interface.c`):** Inicialização do ncurses e renderização do mapa no terminal do cliente.
* **`utils` (`utils.h` / `core/utils.c`):** Definições e funções utilitárias compartilhadas entre cliente e servidor.

<p align="right">(<a href="#readme-top">voltar ao topo</a>)</p>

---

## 📁 Estrutura do Projeto

```
PacketMan/
├── assets/
│   ├── files/               arquivos das pastilhas (1.txt, 2.txt, 3.jpg, 4.jpg, 5.mp4, 6.mp4)
│   ├── map_default/         mapa padrão em formato .csv
│   └── download/            diretório criado em tempo de execução para salvar arquivos recebidos
├── include/                 interfaces e definições dos módulos (*.h)
│   ├── kermit.h             tipos e assinaturas do protocolo Kermit
│   ├── game.h               lógica de mundo e colisões
│   ├── pacman.h             entidade Pac-Man
│   ├── ghosts.h             entidades fantasmas
│   ├── pellets.h            entidades pastilhas
│   ├── interface.h          interface ncurses (cliente)
│   └── utils.h              definições e utilitários compartilhados
├── src/                     código-fonte em C (*.c)
│   ├── client.c             ponto de entrada do cliente
│   ├── server.c             ponto de entrada do servidor
│   ├── core/                módulos de lógica do jogo
│   │   ├── game.c
│   │   ├── ghosts.c
│   │   ├── interface.c
│   │   ├── pacman.c
│   │   ├── pellets.c
│   │   └── utils.c
│   ├── network/             módulos de comunicação
│   │   └── kermit.c
│   └── obj/                 arquivos-objeto intermediários (*.o) — ignorado pelo git
├── .gitignore
├── makefile                 automação de compilação para cliente e servidor
└── README.md
```

<p align="right">(<a href="#readme-top">voltar ao topo</a>)</p>

---

## 🚀 Instalação

### 📦 Pré-requisitos

É necessário um compilador C com suporte a C99, GNU Make, ncurses e permissão para criar raw sockets (geralmente requer `sudo`). No Ubuntu/Debian:

```sh
sudo apt update
sudo apt install build-essential libncurses-dev -y
```

### 🔧 Compilação

1. Clone o repositório:
   ```sh
   git clone https://github.com/GiuTP/PacketMan.git
   cd PacketMan
   ```

2. Compile os dois executáveis:
   ```sh
   make
   ```
   Os executáveis `client` e `server` serão gerados na raiz do projeto.

### ▶ Execução

> **Atenção:** Raw sockets requerem permissões elevadas. Execute ambos os processos com `sudo`.
> A interface de rede (`eth0`, `enp3s0`, etc.) deve ser passada como argumento.

Opcionalmente, é possível passar um arquivo `.csv` de mapa **antes** da interface de rede para usar um mapa personalizado no lugar do padrão:

**Na máquina do servidor:**
```sh
# Mapa padrão
sudo ./server <interface_de_rede>

# Mapa personalizado
sudo ./server <caminho/para/mapa.csv> <interface_de_rede>
```

**Na máquina do cliente (conectada via cabo Ethernet):**
```sh
sudo ./client <interface_de_rede>
```

### ⚙ Comandos Úteis

| Comando | Descrição |
|---------|-----------|
| `make` | Compila cliente e servidor |
| `make client` | Compila apenas o cliente |
| `make server` | Compila apenas o servidor |
| `make clean` | Remove objetos intermediários e executáveis |

<p align="right">(<a href="#readme-top">voltar ao topo</a>)</p>

---

## 📬 Contato

GiuTP — [github.com/GiuTP](https://github.com/GiuTP)

E-mail — giulianotpt@gmail.com

hassevini — [github.com/hassevini](https://github.com/hassevini)

E-mail - hassevini@gmail.com

Link do projeto: [https://github.com/GiuTP/PacketMan](https://github.com/GiuTP/PacketMan)

<p align="right">(<a href="#readme-top">voltar ao topo</a>)</p>

---

## 🙏 Agradecimentos

* [Best-README-Template](https://github.com/othneildrew/Best-README-Template) — template base deste README

---

<!-- MARKDOWN LINKS & IMAGES -->
[stars-shield]: https://img.shields.io/github/stars/GiuTP/PacketMan.svg?style=for-the-badge
[stars-url]: https://github.com/hassevini/Pacman-Redes/stargazers
[issues-shield]: https://img.shields.io/github/issues/GiuTP/PacketMan.svg?style=for-the-badge
[issues-url]: https://github.com/hassevini/Pacman-Redes/issues
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://www.linkedin.com/in/hassevini
[C-badge]: https://img.shields.io/badge/C-00599C?style=for-the-badge&logo=c&logoColor=white
[C-url]: https://en.wikipedia.org/wiki/C_(programming_language)
[Linux-badge]: https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black
[Linux-url]: https://www.kernel.org/
