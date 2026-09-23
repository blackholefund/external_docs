# Production Change History

*Genese Capital (formerly BlackHole Capital). Last reviewed September 2026.*

This page lists the configuration changes that affect how the trading account history should be read. Performance is published on [Myfxbook](https://www.myfxbook.com/members/blackholeai/blackhole-fund/11784758) and is not reported here.

| Date | Change | Affects |
|------|--------|---------|
| 7 Aug 2024 | Trading starts (proprietary capital) | - |
| 27 Aug 2024 - 14 May 2025 | Migration from previous broker to OnEquity; no trading | All |
| 2 Sep 2025 | First investor deposit | - |
| 22 Sep 2025 | Current configuration: pairs netted via Close By | Signal, execution |
| Oct 2025 | 1% daily loss limit added to the 5% cumulative limit | Risk controls |
| 6 Oct 2025 | 3,000,000 allocation; change of scale | Sizing |
| 2 Jun 2026 | Sizing optimisation: median lot from 17-18 to approx. 5; maximum approx. 7.5 per order | Sizing |

## Phases

- **Phase I (Aug 2024 - Sep 2025):** previous configuration, traded with proprietary capital only. No third-party capital was exposed to this configuration.
- **Phase II (since 22 Sep 2025):** current configuration and the only period representative for new allocations.

## Notes for Reviewers

- **Other instruments.** A small number of early trades outside gold were connection and execution tests on the new broker. Since September 2025 the account trades XAUUSD only.
- **Close By.** Since the current configuration, positions are closed by opening an opposite position and netting both via Close By. MT5 books the whole result of the pair on one leg and zero on the other, so splitting the history by direction does not reflect economic attribution.
- **Balance movements.** Growth of the account balance is mainly due to investor deposits, including the allocation of 6 October 2025. This is why Myfxbook *Gain* (time-weighted) and *Absolute Gain* differ. Performance fees are booked as balance operations.

Further detail is available in the *Due Diligence Reference*, on request.
