# Smart Contract Security Portfolio

This repository serves as an archive of my security reviews, ranging from community contests to deep-dive shadow audits and foundational security training.

---

## 🛡️ Audit Portfolio

### 1. Competitive Audits (Sherlock, Codehawks, Code4rena)
*Reports from live competitive bug bounties.*
| Protocol | Platform | Date | Status/Report | High/Medium |
| :--- | :--- | :--- | :--- | :--- |
| [Coming Soon] | [Coming Soon] | [Coming Soon] | [Coming Soon] | [Coming Soon] |

### 2. Shadow Audits
*Independent reviews of live protocols or concluded contests to test zero-knowledge vulnerability discovery.*
| Protocol | Focus Area | Date | Report | Key Findings Focus |
| :--- | :--- | :--- | :--- | :--- |
| **Symmio** | Derivatives, Perpetual Logic, Rounding Errors | [Coming Soon] | [Link to Report] | Precision Loss, Vesting Logic |

### 3. Community Challenges & First Flights
*Foundational protocol reviews and structured security training (e.g., Cyfrin Updraft).*
| Project | Type | Date | Report |
| :--- | :--- | :--- | :--- |
| **TSwap** | Cyfrin Updraft | [2026-04] | [Link to Report](https://github.com/akmalzufaaaf/sc-audit-portfolio/blob/main/2026-26-04-TSwap-Audit.md) |
| **Puppy Ruffle** | Cyfrin Updraft | [2026-03] | [Link to Report](https://github.com/akmalzufaaaf/sc-audit-portfolio/blob/main/2026-03-03-PuppyRuffle-audit.pdf) |
| **Password Store** | Cyfrin Updraft | [2026-02] | [Link to Report](https://github.com/akmalzufaaaf/sc-audit-portfolio/blob/main/2026-16-02-passwordstore-audit.pdf) |

### 4. 🏆 RadCipher Audit Arena
*Recurring competitive audit arena. Weekly placements and overall standing.*

**🏆 All-Time Leaderboard: 27th**

| Week | Date | Placement | Report |
| :--- | :--- | :--- | :--- |
| **Week 15** | [2026-05] | 🏆 3rd Place | [Link to Report](https://github.com/akmalzufaaaf/sc-audit-portfolio/blob/main/2026-15-05-radcipher-audit-arena-week-15.md) |
| **Week 13** | [2026-05] | 🏆 6th Place | [Link to Report](https://github.com/akmalzufaaaf/sc-audit-portfolio/blob/main/2026-6-05-radcipher-audit-arena-week-13.md) |
| **Week 11** | [2026-04] | 🏆 6th Place | [Link to Report](https://github.com/akmalzufaaaf/sc-audit-portfolio/blob/main/2026-16-04-radcipher-audit-arena-week-11.md) |
| **Week 10** | [2026-04] | 🏆 7th Place | [Link to Report](https://github.com/akmalzufaaaf/sc-audit-portfolio/blob/main/2026-9-04-radcipher-audit-arena-week-10.md) |

---

## 🔬 Audit Methodology & Tooling

I do not rely solely on automated scanners. My auditing process is heavily biased towards understanding the macro-economic incentives and business logic of the protocol before looking at the syntax.

1.  **Architecture & Threat Modeling:** Mapping external interactions, role-based access controls, and state transitions.
2.  **Invariant Identification:** Defining the mathematical absolutes of the protocol (e.g., $x \cdot y = k$ in AMMs, proportional share accounting in ERC4626 Vaults).
3.  **Manual Code Review:** Line-by-line inspection focusing on business logic flaws, precision loss, DoS vectors, and oracle manipulation.
4.  **Testing & PoC:** Developing Foundry test suites to mathematically prove exploitability.

**Tools utilized:** `Foundry` (Forge/Cast), `Slither`, `Aderyn`, `Etherscan/Block Explorers`.

---

## 📫 Connect

- **X (Twitter):** [@moal](https://twitter.com/waiwaTF)
- **Medium:** [@moaludin](https://medium.com/@moaludin)
- **LinkedIn:** [Akmal Zufa Fathurrahman](https://linkedin.com/in/akmalzufafathurrahman)

> *"Security is not a feature; it's the absolute foundation."*
