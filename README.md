# Online Stock Trading Platform

A multi-client, real-time stock trading system written in C++ over TCP. A central server holds the market and every account, and clients connect over a LAN to buy and sell stocks through a Dear ImGui desktop interface.

Built as a team project for **CSD2161 Computer Networks** at DigiPen Institute of Technology (2026).

## Features

- **Server-authoritative trading.** The server validates every buy and sell order, updates balances and holdings, and moves prices.
- **Market-maker pricing.** Each stock has a bid/ask spread. A buy pushes the price up by $1 and a sell pushes it down by $1 (minimum $1).
- **Live market broadcast.** Every trade broadcasts the new market state to all connected clients.
- **Custom binary protocol.** Big-endian messages with a 1-byte command ID, plus `recvAll` / `sendExact` helpers that handle partial reads and writes.
- **Concurrent server.** A producer-consumer thread pool of 10 workers, with shared state guarded by mutexes.
- **Persistence.** Users, market data and a timestamped trade log are saved to text files and survive restarts.
- **GUI client.** Dear ImGui on DirectX 11, with a login screen, market table, price graph, portfolio, buy/sell panels, trade history and a colour-coded activity log.

## Tech stack

C++20 · Winsock2 (TCP) · Dear ImGui · DirectX 11 · MSBuild / Visual Studio 2022 · Python (load-testing bot)

## Repository layout

```
├── Client/              ImGui client (Client.cpp, ClientUI.cpp/.h, vendored imgui/)
├── Server/              Trading server, thread pool, data files
│   ├── Stocks.txt       Market data: Symbol Quantity Bid Ask
│   ├── Users.txt        Accounts:    Username Cash [Symbol Qty Cost ...]
│   └── Trades.txt       Trade audit log
├── tools/
│   └── mock_trades.py   Bot script that runs random trades to exercise the server
├── docs/
│   └── Design_Report.pdf
├── CSD2161_A5.sln
├── build.bat            Command-line build (Release/Debug, x64)
└── clean.bat            Removes build artifacts
```

## Building

**Requirements:** Windows 10/11, Visual Studio 2022 (or the 2022 Build Tools) with the *Desktop development with C++* workload and the Windows 10/11 SDK.

**Command line:**

```bat
build.bat
```

Pick `1` for Release (GUI only) or `2` for Debug (GUI plus a console window). The executables are written to `x64\<Config>\`.

**Visual Studio:** open `CSD2161_A5.sln`, select `x64` and `Release` or `Debug`, then build the solution.

A post-build step copies `Stocks.txt`, `Users.txt` and `Trades.txt` next to `Server.exe`.

## Running

```bat
cd x64\Release
Server.exe        :: enter a port, e.g. 8888
Client.exe        :: enter server IP, port and a username
```

New users start with $10,000. Open several clients, on the same machine or on other machines on the LAN, to see market updates spread between them.

To generate trading activity automatically:

```bat
python tools\mock_trades.py 127.0.0.1 8888
```

## Protocol

Every message starts with a 1-byte command ID followed by fixed-width fields in network byte order.

| ID     | Command            | Payload                                                    |
|--------|--------------------|------------------------------------------------------------|
| `0x01` | `REQ_QUIT`         | none                                                       |
| `0x02` | `REQ_LOGIN`        | NameLen(4) + Username(N)                                   |
| `0x03` | `RSP_LOGIN`        | Balance(4)                                                 |
| `0x04` | `REQ_BUY`          | Symbol(8) + Qty(4)                                         |
| `0x05` | `RSP_BUY`          | Symbol(8) + Qty(4) + UnitPrice(4) + NewCash(4)             |
| `0x06` | `REQ_SELL`         | Symbol(8) + Qty(4)                                         |
| `0x07` | `RSP_SELL`         | Symbol(8) + Qty(4) + UnitPrice(4) + NewCash(4)             |
| `0x08` | `REQ_PORTFOLIO`    | none                                                       |
| `0x09` | `RSP_PORTFOLIO`    | Cash(4) + Count(4) + [Symbol(8) + Qty(4) + Cost(4)]...     |
| `0x10` | `REQ_HISTORY`      | none                                                       |
| `0x11` | `RSP_HISTORY`      | Len(4) + Text(N)                                           |
| `0x12` | `RSP_MARKET`       | Count(4) + [Symbol(8) + Qty(4) + Bid(4) + Ask(4)]...       |
| `0x13` | `RSP_MARKETUPDATE` | same as `RSP_MARKET`, broadcast to all clients             |
| `0x14` | `REQ_MARKET`       | none                                                       |
| `0x25` | `RSP_ERROR`        | Len(4) + ErrorText(N)                                      |

## Architecture

**Server**
- The listener thread accepts connections and hands each client to a `TaskQueue` worker (10-thread pool).
- Shared state is protected by `UsersMutex`, `MarketMutex` and `g_clientsMutex`.
- User and market state is saved on every client disconnect, and each trade is appended to the trade log.

**Client**
- The main thread runs the ImGui render loop and sends commands.
- A dedicated receiver thread handles server responses and updates mutex-protected shared state.

See [`docs/Design_Report.pdf`](docs/Design_Report.pdf) for the full design write-up.

## Team

Ban Kai Wei Benjamin · Chia Wei Xuan Rachael · Fam Qian Wei Jonathan · Zong Han Law · Muhammad Raziq

### My contributions (Zong Han Law)

**Client UI (Dear ImGui + DirectX 11)**
- Designed and built the graphical client that replaced the original console workflow ([`Client/ClientUI.cpp`](Client/ClientUI.cpp), [`Client/ClientUI.h`](Client/ClientUI.h)).
- Login screen for the server address, port and username.
- Live market table that colours bid and ask prices and refreshes when the server broadcasts an update.
- Bid/ask price trend graph for each stock, with hover readouts.
- Trading panel with collapsible sections for the portfolio, buy, sell and trade history.
- Top bar with the username and cash balance, a disconnect button, and a colour-coded activity log for trades, errors and notifications.

**Test tooling**
- Wrote [`tools/mock_trades.py`](tools/mock_trades.py), a Python load-testing script that speaks the server's binary protocol. It connects five bot traders on separate threads, each placing random buy and sell orders. This exercised the server under concurrent load and generated realistic price movement for the graphs.

## AI declaration

This project was built with the assistance of AI, specifically [Claude Code](https://claude.com/claude-code) by Anthropic. AI tools helped with writing code, debugging and documentation. The team reviewed, tested and integrated all AI-assisted output and is responsible for the final work.

## Acknowledgements

- [Dear ImGui](https://github.com/ocornut/imgui), MIT License (`Client/imgui/LICENSE.txt`)
