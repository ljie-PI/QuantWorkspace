# Backtest engine maintenance, licensing, and architecture options

**Research date:** 2026-09-03
**Status:** AFK research artifact; not a final engine-selection decision
**Target context:** an internal-use Python system running daily and 1-minute Alpha, deterministic historical replay, custom mainland China A-share rules, and a system-owned OMS and ledger.

## Scope and evidence rules

This note compares Backtrader with Zipline Reloaded, NautilusTrader, Qlib's backtest path, vectorbt community edition, LEAN, vn.py, RQAlpha, and a deliberately small native event loop. It uses only primary sources: project repositories and release APIs, first-party documentation, source code, and authoritative license text.

Three kinds of statements are kept separate:

- **Verified fact** means a cited first-party source directly supports the statement.
- **Design implication / inference** means the statement follows from the verified architecture but is not a project maintainer's claim.
- **Unknown** means the available official evidence is insufficient and a proof-of-concept or legal review is still required.

Performance numbers are not ranked across projects unless workloads and boundaries are comparable. No common official benchmark was found that replays the same A-share minute universe, corporate actions, order load, matching rules, and ledger assertions across these engines.

## Executive observations without a selection

- Backtrader is a mature bar-event API, but its published package and default-branch commit feed remain at `1.9.78.123` from April 2023, and official package classifiers stop at Python 3.7 ([PyPI metadata](https://pypi.org/pypi/backtrader/json), [default-branch commit feed](https://github.com/mementum/backtrader/commits/master.atom), [version commit](https://github.com/mementum/backtrader/commit/b853d7c90b6721476eb5a5ea3135224e33db1f14)). The repository is not archived, but that fact alone does not establish active maintenance ([repository API](https://api.github.com/repos/mementum/backtrader)). Its GPLv3-or-later license and broker-owned cash/order/position state are material engineering constraints.
- Zipline Reloaded has a modern Apache-2.0 package baseline for Python 3.10-3.13, minute and daily event streams, versioned data-bundle ingestion, and custom calendars, but its blotter, metrics tracker, and ledger are central state owners ([3.1.1 release API](https://api.github.com/repos/stefan-jansen/zipline-reloaded/releases/latest), [package metadata](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/pyproject.toml), [simulation loop](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/src/zipline/gens/tradesimulation.py), [ledger](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/src/zipline/finance/ledger.py)).
- NautilusTrader supplies the strongest first-party evidence for deterministic event processing and current end-to-end engine benchmarks. It is also a full trading kernel whose cache, execution engine, risk engine, and portfolio own substantial state, and its current v2 development package requires Python 3.12-3.14 ([architecture](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/docs/concepts/architecture.md), [v1.231.0 release](https://github.com/nautechsystems/nautilus_trader/releases/tag/v1.231.0), [v2 package metadata](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/python/pyproject.toml)).
- Qlib's backtest path has directly relevant A-share configuration surfaces: a documented 100-share trade unit, directional price limits, suspension treatment, volume limits, costs, and nested daily/intraday execution. Its `Account` and shared `Position` objects nevertheless remain authoritative during execution ([Exchange source](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/qlib/backtest/exchange.py), [Account source](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/qlib/backtest/account.py), [nested execution design](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/docs/component/highfreq.rst)).
- vectorbt community edition is maintained and optimized for array traversal and research sweeps, but its current license is **Apache 2.0 with Commons Clause**, not unmodified Apache-2.0 or OSI-style permissive OSS. Its simulations update cash and asset balances and construct their own `Portfolio` state ([license](https://github.com/polakowo/vectorbt/blob/34b6d5935e3ea3eccd549e2592bc0f455b8045f5/LICENSE.md), [portfolio workflow](https://vectorbt.dev/api/portfolio/base/#vectorbt.portfolio.base.Portfolio.from_order_func)).
- LEAN is an actively developed Apache-2.0 .NET/C# engine with Python algorithm hosting. It is highly modular but owns portfolio, transactions, data, scheduling, fill, fee, slippage, buying-power, and settlement models, so adopting it is closer to adopting a platform than importing a Python library ([repository](https://api.github.com/repos/QuantConnect/Lean), [engine documentation](https://www.quantconnect.com/docs/v2/writing-algorithms/key-concepts/algorithm-engine), [reality models](https://www.quantconnect.com/docs/v2/writing-algorithms/reality-modeling/key-concepts)).
- vn.py is credible for A-share connectivity and data integration, with official XTP, TORA, OST, and EMT gateway references plus RQData and XtQuant integrations. The core `OmsEngine` stores orders, trades, positions, accounts, and active-order caches, while backtest and portfolio capabilities are spread across companion repositories ([official README](https://github.com/vnpy/vnpy/blob/fa5206fe63836f3f8cd1ebd7168fbd19a5e2ff09/README_ENG.md), [core engine](https://github.com/vnpy/vnpy/blob/fa5206fe63836f3f8cd1ebd7168fbd19a5e2ff09/vnpy/trader/engine.py)).
- RQAlpha provides the most explicit built-in A-share simulation rules among the reviewed full backtest engines, including stock T+1, price-limit checks, volume constraints, and minute matching modes. Its repository license, however, explicitly requires Ricequant authorization for use by a legal entity for any purpose; that text conflicts with the simpler Apache-2.0 field in `pyproject.toml`, and the repository license says its terms prevail ([license](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/LICENSE), [package metadata](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/pyproject.toml)).
- A small native event loop can preserve a single system-owned OMS/ledger and the narrowest replacement seam. The trade-off is that calendar semantics, corporate actions, matching, A-share rules, diagnostics, persistence, and replay validation all become internal engineering obligations.

## Comparison table

Replacement cost is an **inference**, not a measured project fact. The rubric is:

- **Low:** engine is a non-authoritative research sidecar behind stable arrays or event records.
- **Medium:** custom ingestion and matching adapters are required, but strategies and accounting can remain system-owned.
- **High:** engine strategy lifecycle, data model, order model, and accounting become embedded.
- **Very high:** a runtime/platform boundary and broad engine-owned state must also be replaced.

| Option | Maintenance and Python compatibility — verified | License — verified | Architecture and event extension — verified | Official minute-performance evidence | A-share customization | Ledger/state integration implication — inference | Replacement cost — inference |
|---|---|---|---|---|---|---|---|
| **Backtrader** | Published `1.9.78.123` on 2023-04-19; the default-branch commit feed ends at the same version commit. No `Requires-Python` is declared, and classifiers list Python 3.2-3.7. The repository is unarchived, but no newer default-branch commit or package release is shown ([PyPI](https://pypi.org/pypi/backtrader/json), [setup.py](https://github.com/mementum/backtrader/blob/b853d7c90b6721476eb5a5ea3135224e33db1f14/setup.py), [commit feed](https://github.com/mementum/backtrader/commits/master.atom), [repository API](https://api.github.com/repos/mementum/backtrader)). | GPLv3-or-later ([license](https://github.com/mementum/backtrader/blob/b853d7c90b6721476eb5a5ea3135224e33db1f14/LICENSE)). | `Cerebro` synchronizes feeds, invokes the broker, then strategy `next`; users can replace the broker and add data, observers, analyzers, commission models, and volume fillers ([Cerebro](https://www.backtrader.com/docu/cerebro/), [broker](https://www.backtrader.com/docu/broker/)). | A 2019 first-party test processed 2M synthetic 15-minute candles. The trading-enabled PyPy run reported 12,743 candles/s; it is not a current CPython or A-share benchmark ([benchmark](https://www.backtrader.com/blog/2019-10-25-on-backtesting-performance-and-out-of-memory/on-backtesting-performance-and-out-of-memory/)). | Generic broker, filler, commission, feed, calendar, and strategy hooks exist. No first-party mainland-China rule package was identified in the reviewed official sources. | Default broker owns cash, value, open orders, fills, and positions. Keeping a separate authoritative system ledger creates either broker replacement work or dual-state reconciliation. | **Medium initially; high after deep broker/A-share customization.** |
| **Zipline Reloaded** | Latest release `3.1.1`, 2025-07-23, added Python 3.13 and NumPy 2 compatibility. Metadata requires Python `>=3.10` and lists 3.10-3.13; the default-branch feed shows a later commit on 2025-11-13 ([release API](https://api.github.com/repos/stefan-jansen/zipline-reloaded/releases/latest), [pyproject](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/pyproject.toml), [commit feed](https://github.com/stefan-jansen/zipline-reloaded/commits/main.atom)). | Apache-2.0 ([license](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/LICENSE)). | Minute/daily clock actions drive transactions, commissions, `handle_data`, and metrics; custom bundles and trading calendars are documented ([loop](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/src/zipline/gens/tradesimulation.py), [bundles](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/docs/source/bundles.rst), [calendars](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/docs/source/trading-calendars.rst)). | No comparable first-party end-to-end minute throughput result was identified in the reviewed release, docs, and source materials. | A custom bundle/calendar plus restrictions, commission, and slippage can model part of the market. No turnkey official A-share rule set was identified. | Blotter and ledger update transactions, orders, cash, portfolio, dividends, splits, and positions before/around strategy callbacks. External ownership means replacing or reconciling core components. | **High** if bundle, Algorithm API, and ledger become embedded. |
| **NautilusTrader** | Latest formal release is `v1.231.0`, 2026-08-02; the release describes the v1-to-v2 Rust/PyO3 cutover. Current v2 metadata declares `2.0.0rc5`, Python `>=3.12,<3.15`, and Python 3.12-3.14 classifiers ([release API](https://api.github.com/repos/nautechsystems/nautilus_trader/releases/latest), [release notes](https://github.com/nautechsystems/nautilus_trader/releases/tag/v1.231.0), [pyproject](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/python/pyproject.toml)). | LGPL-3.0-only ([license](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/LICENSE)). | Rust-native event-driven kernel with message bus, data, risk, execution, cache, portfolio, actors, strategies, and adapters. Backtest, sandbox, and live use the common kernel ([architecture](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/docs/concepts/architecture.md)). v1.231.0 also documents Python-subclassable v2 fill and fee models. | Canonical 2026 benchmark uses 10,000 one-minute bars plus 10,000 quotes and reports about 23-47 ms depending on scenario and timed boundary on a 64-core Threadripper; the benchmark verifies event counts and canonical result digests ([benchmark](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/crates/backtest/benches/BENCHMARKS.md)). | Rich instrument, fill, fee, risk, and adapter surfaces. The official adapter list contains no mainland-China venue adapter ([adapters](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/ADAPTERS.md)). | Kernel explicitly owns cache, account/order/position state, execution lifecycle, risk, and derived portfolio state. Subordinating it to an external ledger cuts across normal strategy APIs. | **Very high** as the main kernel; potentially **medium/high** if isolated to a narrow replay/matching service. |
| **Qlib backtest path** | Latest formal release `v0.9.7`, 2025-08-15; the default-branch feed shows a later commit on 2026-07-23. Metadata requires Python `>=3.8` and lists 3.8-3.12 ([release feed](https://github.com/microsoft/qlib/releases.atom), [commit feed](https://github.com/microsoft/qlib/commits/main.atom), [pyproject](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/pyproject.toml)). | MIT ([license](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/LICENSE)). | Strategy, executor, exchange, and account composition supports nested daily and intraday decisions ([nested execution design](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/docs/component/highfreq.rst), [executor](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/qlib/backtest/executor.py)). | Official sources confirm 1-minute data and nested high-frequency execution, but no current end-to-end engine-throughput benchmark comparable to Nautilus or Backtrader was identified ([README](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/README.md), [high-frequency design](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/docs/component/highfreq.rst)). | Direct surfaces for 100-share trade units, directional limits, suspension via missing close, configurable buy/sell volume limits, open/close/minimum costs, and market impact ([Exchange](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/qlib/backtest/exchange.py)). | `Account.update_order` mutates `current_position`; nested accounts shallow-copy while sharing the position object. An external authoritative ledger would need a deliberate replacement or projection boundary. | **Medium** as a research/backtest component; **high** if nested executor/account semantics become core application APIs. |
| **vectorbt community edition** | Latest release `v1.1.0`, 2026-07-05; the default-branch feed shows a later commit on 2026-08-02. Metadata requires Python `>=3.11,<3.15` and the release says wheels are built/tested for 3.11-3.14 ([release API](https://api.github.com/repos/polakowo/vectorbt/releases/latest), [commit feed](https://github.com/polakowo/vectorbt/commits/master.atom), [pyproject](https://github.com/polakowo/vectorbt/blob/34b6d5935e3ea3eccd549e2592bc0f455b8045f5/pyproject.toml)). | Apache 2.0 with Commons Clause; the clause withholds the right to “Sell” products or services whose value derives entirely or substantially from the software ([license](https://github.com/polakowo/vectorbt/blob/34b6d5935e3ea3eccd549e2592bc0f455b8045f5/LICENSE.md)). Earlier reviewed tags such as `v0.21.0` and `v0.28.5` use the same license form ([v0.21.0 license](https://github.com/polakowo/vectorbt/blob/v0.21.0/LICENSE.md), [v0.28.5 license](https://github.com/polakowo/vectorbt/blob/v0.28.5/LICENSE.md)). | Broadcast arrays feed Numba or optional Rust kernels. `from_order_func` provides callback stages and a flexible mode allowing multiple orders per symbol/bar ([portfolio docs](https://vectorbt.dev/api/portfolio/base/#vectorbt.portfolio.base.Portfolio.from_order_func)). | Official benchmark artifacts are correctness-aware Numba/Rust **kernel microbenchmarks**, not complete minute-bar OMS/ledger replays ([benchmark README](https://github.com/polakowo/vectorbt/blob/34b6d5935e3ea3eccd549e2592bc0f455b8045f5/benchmarks/README.md), [Numba matrix](https://github.com/polakowo/vectorbt/blob/34b6d5935e3ea3eccd549e2592bc0f455b8045f5/benchmarks/BENCHMARKS_NUMBA.md)). | Rules can be encoded in input arrays and callbacks. No official mainland-China calendar, T+1, board-lot, auction, or corporate-action rule pack was identified. | Simulation itself fills/rejects orders, updates cash/assets, and creates a `Portfolio`. It fits best with a system ledger when used as a non-authoritative research sidecar or when results are projected into a canonical system event model. | **Low** as a sidecar; **medium/high** if callback state and `Portfolio` records become authoritative. |
| **LEAN** | The default-branch feed shows active development through 2026-09-01. The latest GitHub “Release” object is from 2017 and is not representative of maintenance; current source/CI activity is the useful evidence ([commit feed](https://github.com/QuantConnect/Lean/commits/master.atom), [release API](https://api.github.com/repos/QuantConnect/Lean/releases/latest)). The official foundation container pins a Python 3.11 runtime and uses Python.NET environment wiring ([Dockerfile](https://github.com/QuantConnect/Lean/blob/f24fc0d3df03d6bdbe0e6fc7b8522445f1d900d2/DockerfileLeanFoundation)). | Apache-2.0 ([license](https://github.com/QuantConnect/Lean/blob/f24fc0d3df03d6bdbe0e6fc7b8522445f1d900d2/LICENSE)). | Event-driven .NET engine with Python algorithm support; components and reality models are pluggable/customizable ([README](https://github.com/QuantConnect/Lean/blob/f24fc0d3df03d6bdbe0e6fc7b8522445f1d900d2/readme.md), [reality models](https://www.quantconnect.com/docs/v2/writing-algorithms/reality-modeling/key-concepts)). | Regression and test infrastructure is visible, but no first-party benchmark matching the reviewed A-share minute workload was identified. | Custom data and fee/fill/slippage/buying-power/settlement models provide broad extension points. No official turnkey mainland-China equities data/brokerage path was identified in the reviewed sources. | LEAN's algorithm engine manages portfolio, transactions, securities, data feeds, and scheduling. External OMS/ledger ownership means bypassing or continuously reconciling core engine state. | **Very high** because of the .NET/Python boundary, LEAN data/model formats, and broad internal state. |
| **vn.py** | Latest release `4.4.0`, 2026-05-14; the default-branch feed shows a later commit on 2026-08-06. Metadata requires Python `>=3.10` and lists 3.10-3.13 ([release API](https://api.github.com/repos/vnpy/vnpy/releases/latest), [commit feed](https://github.com/vnpy/vnpy/commits/master.atom), [pyproject](https://github.com/vnpy/vnpy/blob/fa5206fe63836f3f8cd1ebd7168fbd19a5e2ff09/pyproject.toml)). | MIT ([license](https://github.com/vnpy/vnpy/blob/fa5206fe63836f3f8cd1ebd7168fbd19a5e2ff09/LICENSE)). | A threaded typed event queue supports per-type and general handlers; `MainEngine` loads gateways and application engines ([event engine](https://github.com/vnpy/vnpy/blob/fa5206fe63836f3f8cd1ebd7168fbd19a5e2ff09/vnpy/event/engine.py), [main engine](https://github.com/vnpy/vnpy/blob/fa5206fe63836f3f8cd1ebd7168fbd19a5e2ff09/vnpy/trader/engine.py)). | No comparable official minute-bar throughput benchmark was identified. | Strong first-party ecosystem evidence for A-share gateways, data services, and examples; portfolio/backtest applications are separate modules ([README](https://github.com/vnpy/vnpy/blob/fa5206fe63836f3f8cd1ebd7168fbd19a5e2ff09/README_ENG.md)). | Core `OmsEngine` stores order, trade, position, account, contract, quote, and active-order state. Using only gateway/event abstractions is less coupled than adopting the full OMS/backtester stack. | **Medium/high** for selected connectivity modules; **high** for the full platform. |
| **RQAlpha** | Latest release `release/6.3.0`, 2026-07-23; the default-branch feed shows a later commit on 2026-09-01. Metadata requires Python `>=3.8` and lists 3.8-3.14 ([release API](https://api.github.com/repos/ricequant/rqalpha/releases/latest), [commit feed](https://github.com/ricequant/rqalpha/commits/master.atom), [pyproject](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/pyproject.toml)). | Custom Ricequant license; legal entities require authorization for any use, and the custom text prevails over conflicting Apache terms ([license](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/LICENSE)). | Synchronous event-source executor emits pre/main/post phases for bars, ticks, auctions, session transitions, and settlement; system behavior is organized into Mods ([executor](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/rqalpha/core/executor.py), [event definitions](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/rqalpha/core/events.py)). | No comparable official throughput benchmark was identified. | Built-in stock T+1, position validation, delisting cash behavior, current/next-bar and VWAP minute matching, price-limit checks, inactivity checks, and volume participation controls ([accounts Mod](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/rqalpha/mod/rqalpha_mod_sys_accounts/README.rst), [simulation Mod](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/rqalpha/mod/rqalpha_mod_sys_simulation/README.rst)). | Account, position, matching, risk, and costs are system Mods. An external ledger requires replacing/subordinating those modules or reconciling them. | **High technically**, with a separate **license-authorization gate** before corporate evaluation. |
| **Small native event loop** | Maintained against the system's chosen Python baseline; no third-party release dependency. | Internal code under the organization's chosen policy. | Ordered immutable events plus explicit market-calendar, A-share policy, matcher, OMS, and ledger interfaces. | No evidence exists until a representative benchmark and deterministic replay test suite are built. | Full control, but every rule must be implemented and validated internally. | Can make the system OMS and ledger the only authoritative state owners. | **High initial construction**, potentially **low later engine-replacement cost** if interfaces remain narrow. |

## Licensing engineering notes

This section is engineering information, **not legal advice**. The exact deployment, recipients, contractors, affiliates, packaging, modifications, and distribution model should be reviewed by counsel.

### Backtrader and GPLv3-or-later

**Verified facts**

- Backtrader declares GPLv3-or-later in both package metadata and source headers ([setup.py](https://github.com/mementum/backtrader/blob/b853d7c90b6721476eb5a5ea3135224e33db1f14/setup.py), [license](https://github.com/mementum/backtrader/blob/b853d7c90b6721476eb5a5ea3135224e33db1f14/LICENSE)).
- GPLv3 section 2 says covered works may be made, run, and propagated without conditions when they are not conveyed. Sections 5 and 6 impose conditions when modified source or object-code covered works are conveyed ([GPLv3 authoritative text](https://www.gnu.org/licenses/gpl-3.0.txt)).

**Design implications / inferences**

- Purely private internal execution may have a materially different compliance profile from shipping software to customers or other recipients, but the boundary of “conveying,” the identity of the relevant legal entity, contractor arrangements, and whether the combined application is a covered work are legal questions.
- An in-process `Adapter` is an architectural abstraction, not a license exception. This research does **not** claim that wrapping Backtrader behind an Adapter removes or limits GPL obligations.
- A separate process or service boundary may improve replacement and state isolation, but it should not be represented as automatically changing the license outcome.

**Unknowns**

- Whether the intended organizational/deployment topology constitutes conveying.
- Whether modifications would be made to Backtrader itself.
- What source, notice, installation, or recipient obligations would apply to the actual combined deployment.

### NautilusTrader and LGPLv3-only

**Verified facts**

- NautilusTrader's package metadata declares LGPL-3.0-only ([pyproject](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/python/pyproject.toml)).
- LGPLv3 defines an “Application” and “Combined Work” and, when a Combined Work is conveyed, section 4 requires notices and either corresponding application code suitable for relinking or a suitable shared-library mechanism, among other conditions ([LGPLv3 authoritative text](https://www.gnu.org/licenses/lgpl-3.0.txt)).

**Design implications / inferences**

- LGPL is materially different from Backtrader's GPL for library integration, but Python bindings, Rust/PyO3 packaging, modifications, container distribution, and deployment topology still need an explicit compliance design.
- Treating NautilusTrader as a separately deployed service can reduce application coupling, but is not a substitute for legal review.

### Apache-2.0 and MIT candidates

Zipline Reloaded and LEAN include Apache-2.0 license text ([Zipline license](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/LICENSE), [LEAN license](https://github.com/QuantConnect/Lean/blob/f24fc0d3df03d6bdbe0e6fc7b8522445f1d900d2/LICENSE)). Qlib and vn.py include MIT license text ([Qlib license](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/LICENSE), [vn.py license](https://github.com/vnpy/vnpy/blob/fa5206fe63836f3f8cd1ebd7168fbd19a5e2ff09/LICENSE)).

**Design implication / inference:** these licenses are generally less restrictive for proprietary integration than GPL-family licenses, but redistribution still requires preserving the applicable notices and terms. Dependency licenses must be reviewed separately.

### vectorbt community edition

**Verified facts**

- Current vectorbt community edition is “Apache 2.0 with Commons Clause.” The Commons Clause withholds the right to sell the software as defined in the license, including certain paid hosted, consulting, or support offerings whose value derives substantially from vectorbt ([current license](https://github.com/polakowo/vectorbt/blob/34b6d5935e3ea3eccd549e2592bc0f455b8045f5/LICENSE.md)).
- The same license form is present in reviewed historical tags, so “vectorbt OSS” should not be used as shorthand for plain Apache-2.0 without qualification ([v0.21.0 license](https://github.com/polakowo/vectorbt/blob/v0.21.0/LICENSE.md), [v0.28.5 license](https://github.com/polakowo/vectorbt/blob/v0.28.5/LICENSE.md)).

**Design implications / inferences**

- Internal research use is not the same scenario as selling a vectorbt-derived product or service, but productization, client reporting services, hosted research, and paid support scenarios need license review.
- The correct engineering label for the current community edition is “source-available under Apache 2.0 with Commons Clause,” not simply “permissive OSS.”

### RQAlpha

**Verified facts**

- `pyproject.toml` says Apache-2.0, but the repository `LICENSE` defines non-commercial use narrowly and states that any legal entity or other organization needs authorization for any purpose; it also says the custom license prevails where it conflicts with Apache-2.0 ([pyproject](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/pyproject.toml), [license](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/LICENSE)).

**Design implication / inference:** corporate internal use should be treated as blocked pending written authorization and terms, regardless of the package metadata classifier.

## Detailed architecture, state ownership, and fit implications

### Backtrader

**Verified facts**

- `Cerebro` is the central orchestration object for data feeds, strategies, observers, analyzers, writers, broker execution, and results ([Cerebro docs](https://www.backtrader.com/docu/cerebro/)).
- Its documented loop delivers bars, notifies strategies of broker events, asks the broker to execute pending orders, and then calls strategy `next`. The docs explicitly say delivered bars are closed and normal orders are first eligible on the following bar (`x + 1`) ([Cerebro backtesting logic](https://www.backtrader.com/docu/cerebro/)).
- `BackBroker` checks cash, maintains cash/value and positions, supports multiple order types, and offers custom commission, slippage, and volume-filler hooks ([broker docs](https://www.backtrader.com/docu/broker/)).

**Design implications / inferences**

- Backtrader is straightforward for daily/bar strategies and custom indicators.
- Deterministic replay is possible only if every data ordering rule, strategy input, random seed, custom filler, and broker extension is controlled. The official docs do not provide a canonical output-digest protocol.
- A-share T+1 sell availability, board-lot rounding, board/ST-specific price bands, auctions, suspensions, delisting, stamp duty, transfer fees, and corporate actions would be application-specific work. Implementing those inside `BackBroker` increases replacement cost and GPL review scope.
- If the system OMS assigns order IDs and the system ledger is authoritative, the cleanest Backtrader use would avoid strategy decisions depending directly on Backtrader broker balances/positions. That gives up some convenience and requires explicit projection/reconciliation.

**Unknowns**

- Compatibility with the project's exact production Python, NumPy, pandas, and plotting stack.
- Throughput and memory for the actual A-share universe and order workload.
- How much existing Backtrader-dependent strategy code would need migration.

### Zipline Reloaded

**Verified facts**

- The simulation clock emits session, bar, and minute-end actions. At each bar the blotter creates transactions and commissions; the metrics tracker processes them before `handle_data` runs ([simulation loop](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/src/zipline/gens/tradesimulation.py)).
- A data bundle includes pricing, adjustments, and an asset database. Multiple timestamped ingestions are retained so older backtests can reuse the data available at an earlier ingestion date ([bundle docs](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/docs/source/bundles.rst)).
- Bundle writers accept minute bars, daily bars, splits, mergers, dividends, and stock dividends; custom calendars define exchange time zones, sessions, opens/closes, holidays, and special opens/closes ([bundle docs](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/docs/source/bundles.rst), [calendar docs](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/docs/source/trading-calendars.rst)).
- `Ledger` describes itself as tracking all orders and transactions plus current portfolio and position state; transaction processing changes cash and positions ([ledger source](https://github.com/stefan-jansen/zipline-reloaded/blob/943010b9da848e317fc520de87edade2b884d329/src/zipline/finance/ledger.py)).

**Design implications / inferences**

- Versioned bundle ingestion is useful for reproducible research, but it also introduces a Zipline-specific data-format and asset-ID migration surface.
- The event loop is deterministic only if bundle contents, calendar, adjustment data, restrictions, extension registration, and all model parameters are frozen.
- Custom A-share support is technically possible, but the work is broader than a calendar: price-band directionality, T+1 inventory, board lots, auctions, suspension, fees, and delisting must integrate with blotter/ledger behavior.
- Making the system ledger authoritative requires either a custom blotter/metrics path or a strict post-run projection with reconciliation. Strategies that read Zipline portfolio/account APIs would otherwise depend on non-authoritative state.

**Unknowns**

- Current maintainer capacity and future release cadence.
- Actual minute replay performance on the target dataset.
- Corporate-action correctness for mainland-China-specific events and historical rule changes.

### NautilusTrader

**Verified facts**

- NautilusTrader's common kernel includes `DataEngine`, `RiskEngine`, `ExecutionEngine`, `Portfolio`, `Trader`, `MessageBus`, and `Cache`. The cache stores instruments, accounts, orders, and positions; the portfolio tracks balances, positions, margin, PnL, and exposure ([architecture](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/docs/concepts/architecture.md)).
- Backtest, sandbox, and live contexts share the common kernel. The documented execution flow validates a strategy command in the risk engine, routes it through the execution engine/client, and returns accepted/filled/canceled/rejected events that update cache and portfolio state ([architecture](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/docs/concepts/architecture.md)).
- The architecture explicitly identifies deterministic behavior as a quality outcome when ordering and configuration are deterministic, and the canonical benchmark checks exact result digests ([architecture](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/docs/concepts/architecture.md), [benchmark](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/crates/backtest/benches/BENCHMARKS.md)).
- Official adapters do not include a mainland-China venue ([adapter list](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/ADAPTERS.md)).

**Design implications / inferences**

- The architecture is well suited to deterministic replay and event extension, but it duplicates the proposed system-owned OMS/ledger most strongly.
- Using Nautilus as the main kernel would likely require the system to accept Nautilus cache/execution/portfolio objects as authoritative, or maintain a carefully reconciled mirror.
- Using it only for historical matching reduces coupling but may bypass much of the platform value and still requires mapping instruments, orders, fills, fees, and lifecycle events.
- The v2 transition creates near-term API and migration risk even though maintenance evidence is strong.

**Unknowns**

- v2 final API stability and wheel availability on every target host.
- Practical LGPL compliance for the intended packaging and deployment.
- Cost of implementing historical A-share instruments, auctions, T+1, price bands, and corporate actions.
- Whether the benchmark remains representative at thousands of symbols with sparse minute bars and system-ledger projections.

### Qlib backtest path

**Verified facts**

- Qlib describes a nested decision-execution framework in which daily decisions can be decomposed into finer intraday execution decisions, with customizable frequency, decision content, and execution environments ([high-frequency design](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/docs/component/highfreq.rst)).
- `Exchange` documents `trade_unit` as 100 for China A shares, supports separate buy/sell limit expressions, treats missing close as suspension, supports cumulative/current volume limits, and configures open cost, close cost, minimum cost, and impact cost ([Exchange source](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/qlib/backtest/exchange.py)).
- `Account.update_order` changes `current_position`; the account documentation says nested executors use shallow copies while sharing the position object across levels ([Account source](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/qlib/backtest/account.py)).

**Design implications / inferences**

- Qlib is unusually close to the daily-Alpha plus minute-execution problem shape.
- Its A-share support is a useful starting point, not a complete historical exchange-rule specification. T+1 sell inventory, auction phases, ST/board-specific limits, fee changes, rights issues, and delisting details still need verification or extension.
- The shared `Position` design makes it difficult to call the external system ledger authoritative while simultaneously using Qlib strategies that depend on `Account.current_position`.
- A lower-coupling use is to keep Alpha generation and perhaps execution scheduling in Qlib while converting proposed orders/fills into the system's canonical event schema.

**Unknowns**

- Replay determinism under the exact nested daily/minute workflow.
- Current handling of every required A-share corporate action and historical rule regime.
- End-to-end throughput for the target universe.
- Whether account/position mutation can be safely replaced without forking core behavior.

### vectorbt community edition

**Verified facts**

- vectorbt prepares broadcast inputs, traverses the time-by-asset shape in Numba/Rust simulation code, processes orders, updates cash and asset balances, records fills, and constructs a `Portfolio` object ([portfolio workflow](https://vectorbt.dev/api/portfolio/base/#vectorbt.portfolio.base.Portfolio.from_order_func)).
- `from_order_func` exposes callback stages; flexible mode can create multiple orders per symbol and bar ([portfolio docs](https://vectorbt.dev/api/portfolio/base/#vectorbt.portfolio.base.Portfolio.from_order_func)).
- Official benchmark tooling compares deterministic paired Numba and Rust functions and explicitly describes the results as microbenchmarks sensitive to hardware and software environment ([benchmark README](https://github.com/polakowo/vectorbt/blob/34b6d5935e3ea3eccd549e2592bc0f455b8045f5/benchmarks/README.md)).

**Design implications / inferences**

- vectorbt is attractive for wide parameter sweeps and array-native daily/minute research.
- Encoding a full event-sensitive A-share matcher inside callbacks can erode the simplicity and speed benefits that make vectorbt useful.
- A good state-ownership boundary is to treat vectorbt outputs as research evidence or candidate order intents, never as the canonical ledger. If used for execution simulation, canonical system fill and ledger events should be regenerated or rigorously reconciled.

**Unknowns**

- End-to-end speed after T+1 inventory, price limits, board lots, partial fills, suspensions, corporate actions, and ledger reconciliation are added.
- License implications for any future paid service or product path.
- How much callback complexity remains maintainable and testable.

### LEAN

**Verified facts**

- LEAN describes itself as an event-driven, modular algorithmic trading platform with pluggable/customizable components ([README](https://github.com/QuantConnect/Lean/blob/f24fc0d3df03d6bdbe0e6fc7b8522445f1d900d2/readme.md)).
- The engine loads a `QCAlgorithm`, synchronizes requested data, injects data, processes orders, and updates algorithm state. It manages portfolio and data feeds and exposes securities, portfolio, transactions, schedule, notifications, and universe managers ([algorithm engine docs](https://www.quantconnect.com/docs/v2/writing-algorithms/key-concepts/algorithm-engine)).
- Reality models include portfolio, brokerage, fills, slippage, and capacity, and users are directed to create custom models for illiquid or high-volume cases ([reality models](https://www.quantconnect.com/docs/v2/writing-algorithms/reality-modeling/key-concepts)).
- The official foundation container configures Python.NET for Python 3.11 and builds on .NET tooling ([Dockerfile](https://github.com/QuantConnect/Lean/blob/f24fc0d3df03d6bdbe0e6fc7b8522445f1d900d2/DockerfileLeanFoundation), [README build instructions](https://github.com/QuantConnect/Lean/blob/f24fc0d3df03d6bdbe0e6fc7b8522445f1d900d2/readme.md)).

**Design implications / inferences**

- LEAN provides broad extension points but introduces a substantial C#/.NET operational, debugging, and upgrade surface into a Python-owned system.
- Keeping the system OMS/ledger authoritative would require custom integration below the usual `QCAlgorithm` portfolio/transaction abstractions or exact reconciliation against them.
- A-share support would require data conversion, exchange hours, security models, settlement, fees, price bands, and corporate actions; broad generic extension points do not make these rules turnkey.

**Unknowns**

- Target-team cost of Python/.NET debugging and deployment.
- Exact performance for the system's minute data and Python Alpha callbacks.
- Required customization for SSE/SZSE calendars, instruments, and settlement.

### vn.py

**Verified facts**

- `EventEngine` uses a queue and worker thread, dispatches typed events to registered handlers, and supports general handlers ([event engine](https://github.com/vnpy/vnpy/blob/fa5206fe63836f3f8cd1ebd7168fbd19a5e2ff09/vnpy/event/engine.py)).
- `MainEngine` loads gateways and application engines. Its `OmsEngine` registers order, trade, position, account, contract, and quote event handlers and stores those objects in dictionaries, including active-order caches ([trader engine](https://github.com/vnpy/vnpy/blob/fa5206fe63836f3f8cd1ebd7168fbd19a5e2ff09/vnpy/trader/engine.py)).
- Official project documentation lists XTP, TORA, OST, and EMT as domestic A-share gateways; RQData and XtQuant provide market data; separate CTA backtester and portfolio-strategy repositories provide backtest/application functionality ([README](https://github.com/vnpy/vnpy/blob/fa5206fe63836f3f8cd1ebd7168fbd19a5e2ff09/README_ENG.md)).

**Design implications / inferences**

- vn.py is especially credible as a source of A-share connectivity patterns and gateway integration.
- The threaded live event engine is not by itself evidence of deterministic historical replay. A deterministic backtest should use a single ordered scheduler or otherwise control concurrency and timer events.
- Using selected gateways or data adapters behind the system OMS is less coupled than adopting `MainEngine` plus `OmsEngine` plus companion backtester applications.
- Version alignment across core and companion repositories is an ongoing replacement/upgrade concern.

**Unknowns**

- Exact version compatibility across core, portfolio strategy, backtester, data services, and selected A-share gateways.
- Whether companion backtest applications implement all required A-share historical rules.
- Comparable minute throughput and deterministic replay guarantees.

### RQAlpha

**Verified facts**

- `Executor` consumes an event source and synchronously emits before/main/after phases for bars, ticks, open auctions, trading-day boundaries, and settlement ([executor](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/rqalpha/core/executor.py)).
- The accounts Mod has a configurable stock T+1 restriction and stock-position validation ([accounts Mod](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/rqalpha/mod/rqalpha_mod_sys_accounts/README.rst)).
- The simulation Mod documents daily, minute, and tick matching modes; price-limit, liquidity, volume, and inactive-security controls; and custom slippage model classes ([simulation Mod](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/rqalpha/mod/rqalpha_mod_sys_simulation/README.rst)).

**Design implications / inferences**

- The rule set is directly relevant to A-share replay, but the account, matcher, and Mod architecture still competes with a system-owned OMS/ledger.
- The event-phase model is extensible and suitable for deterministic replay if the event source and all Mods are deterministic.
- Corporate use cannot proceed as an ordinary open-source dependency evaluation until authorization terms are obtained.

**Unknowns**

- Authorization availability, cost, deployment restrictions, and modification/redistribution terms.
- Completeness of historical A-share rule changes and corporate actions.
- Throughput on the target minute universe.

## Minute-bar performance interpretation

### What is verified

- Backtrader's 2019 benchmark used 100 synthetic instruments × 20,000 15-minute candles. The article reports multiple execution modes; the trading-enabled PyPy scenario processed 2 million candles at 12,743 candles/s and consumed about 1.3 GB peak memory. The workload intentionally generated many random crossovers and orders ([Backtrader benchmark](https://www.backtrader.com/blog/2019-10-25-on-backtesting-performance-and-out-of-memory/on-backtesting-performance-and-out-of-memory/)).
- NautilusTrader's 2026 canonical benchmark processes 10,000 one-minute bars plus 10,000 derived quotes, validates action/event counts and exact result digests, and reports roughly 23-47 ms for the published scenarios/boundaries on a 64-core AMD Threadripper 9980X ([Nautilus benchmark](https://github.com/nautechsystems/nautilus_trader/blob/b57205b5c2c0b5c43413cf1e42ff5e0a9ed83f35/crates/backtest/benches/BENCHMARKS.md)).
- vectorbt's official matrices time individual Numba and Rust functions with deterministic arrays; the project explicitly frames them as correctness-aware microbenchmarks rather than whole-system portfolio runs ([benchmark README](https://github.com/polakowo/vectorbt/blob/34b6d5935e3ea3eccd549e2592bc0f455b8045f5/benchmarks/README.md)).
- Qlib officially supports 1-minute data and nested intraday execution, but the reviewed official materials do not publish a comparable engine-throughput result ([README](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/README.md), [nested execution](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/docs/component/highfreq.rst)).

### What must not be inferred

- The Nautilus number cannot be divided by the Backtrader number to claim a general speedup. They differ in year, language/runtime, hardware, event count, bar frequency, order count, setup boundary, correctness checks, and data shape.
- vectorbt kernel timings do not establish full-OMS performance with T+1, price bands, partial fills, corporate actions, and external-ledger reconciliation.
- The absence of a published benchmark for Zipline Reloaded, Qlib, LEAN, vn.py, or RQAlpha is not evidence that an engine is slow. It means performance remains unknown for this workload.

### Required internal benchmark

A defensible proof-of-concept should use identical inputs and canonical outputs for every candidate:

1. At least one representative multi-year daily universe.
2. At least one representative A-share 1-minute universe with sparse bars and suspensions.
3. Identical point-in-time corporate actions and instrument metadata.
4. Identical Alpha decisions or order-intent stream.
5. Identical T+1, lot, price-band, auction, volume-participation, fee/tax, and delisting policies.
6. Identical warm/cold boundaries: ingestion excluded and included.
7. Assertions on every order transition, fill, cash movement, settlement bucket, position, and final digest.
8. Wall time, CPU time, peak RSS, data-load time, event throughput, and reconciliation overhead.

## Proposed small native event loop

This is a **design proposal**, not a verified third-party implementation.

### Minimal canonical events

```text
MarketEvent(timestamp, instrument, bar_or_tick)
CorporateActionEvent(timestamp, instrument, action)
SessionEvent(timestamp, phase)
StrategyDecision(timestamp, intents)
OrderCommand(order_id, instrument, side, quantity, constraints)
OrderEvent(order_id, state, reason)
FillEvent(fill_id, order_id, price, quantity, fees)
LedgerEvent(sequence, resulting_balances_and_positions)
```

### Deterministic ordering proposal

```text
(event_time, session_phase, source_priority, source_sequence)
```

Every input event should carry an immutable source sequence. Every output event should be append-only and canonically serializable so replay can compare a final cryptographic digest and, on mismatch, the first divergent sequence.

### Ownership proposal

- The **system OMS** assigns order IDs and owns order lifecycle.
- The **system ledger** alone owns cash, positions, realized PnL, settlement/T+1 availability, fees, taxes, and corporate-action postings.
- The replay loop owns only cursor/scheduling state and pending immutable events.
- Fill models consume a read-only market snapshot plus an order snapshot and emit fill/rejection facts; they do not mutate balances.
- Strategies receive read-only ledger snapshots and submit intents rather than mutating broker objects.
- A-share policy components are independent interfaces: exchange calendar/session phases, auction behavior, board lots, T+1 availability, board/ST price bands, suspension, volume participation, fees/taxes, corporate actions, and delisting.

### Design implications / inferences

- This architecture removes dual-ledger reconciliation by construction.
- It can reduce future engine replacement to converting market/order/fill events at a narrow boundary.
- It has the highest initial validation burden because mature engines' calendars, order semantics, analytics, data ingestion, and diagnostics must be replaced or integrated separately.
- “Small” must refer to the orchestration core, not to the total domain work. A correct A-share simulator remains substantial.

## Unknowns and proof-of-concept gates

### Cross-cutting unknowns

- The production Python baseline and operating systems have not been supplied here; compatibility must be tested against them.
- No candidate has been run against the project's actual data, strategies, or system ledger.
- No official source establishes a complete, historically versioned SSE/SZSE ruleset for all required periods.
- The required scale—instrument count, years, event count, order intensity, and acceptable wall time—must be fixed before performance can be judged.
- Existing strategy and result-format dependencies determine replacement cost but were outside this research scope.

### Candidate-specific gates

- **Backtrader:** install/import/test on the target Python; validate broker replacement and GPL deployment assumptions; measure real minute replay.
- **Zipline Reloaded:** build an A-share bundle/calendar; verify restriction, corporate-action, and T+1 hooks; test whether the ledger can be subordinated.
- **NautilusTrader:** validate v2 API/wheels and LGPL packaging; prototype A-share instrument/matching policies; measure external-ledger projection cost.
- **Qlib:** verify deterministic nested daily/minute execution; test replacing or projecting `Account.current_position`; fill missing historical rule semantics.
- **vectorbt:** implement a limited A-share callback prototype and measure compilation, runtime, memory, and canonical-ledger reconciliation; review intended commercial scenarios.
- **LEAN:** estimate .NET operational cost; prototype custom China market data, calendar, fee/fill/settlement models, and system-ledger synchronization.
- **vn.py:** select exact companion-module versions; distinguish live gateway value from backtest-rule completeness; build a deterministic single-thread replay harness.
- **RQAlpha:** obtain written corporate-use terms before technical adoption work; then validate ledger replacement and historical rule completeness.
- **Native loop:** implement one thin vertical slice with deterministic ordering, one matcher, T+1/lot/price-limit rules, ledger assertions, and a golden replay digest before estimating the full build.

## Decision framing for a later selection

This research does not choose an engine. A later selection should score at least two distinct deployment roles rather than assuming every candidate must be the whole platform:

1. **Research sidecar:** fast Alpha sweeps and approximate portfolio simulation, with the system ledger remaining authoritative.
2. **Deterministic execution simulator:** exact order/fill/replay semantics feeding the system OMS/ledger.
3. **Full trading kernel:** candidate owns risk, execution, account, order, and portfolio state.

The same engine can have a low replacement cost in the first role and a very high cost in the third. The decisive architectural question is therefore not only “Which engine has the most features?” but “Which state is allowed to become authoritative, and which interfaces will strategies be permitted to depend on?”

No final engine recommendation is made here.
