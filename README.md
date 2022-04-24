# Smart Contract Security Testing Guide

[![CC BY-SA 4.0][cc-by-sa-shield]][cc-by-sa]

Smart Contract Security Testing Guide (SCSTG) is a risk-based guide for smart contract security professionals and developers to use as a reference in the security testing of smart contracts. It describes the characteristics and processes for verifying smart contract security issues in different categories, together with examples of vulnerable contracts or functions, and solutions to resolving the risks from their root causes or mitigating their risks.

## Testing Items

The risks are categorized into 9 categories. Each page includes the description of the risks, examples, and solution or mitigations for those risks.

| Category |
|---------------------------|
| [1. Architecture and Design](./testing-items/1-architecture-and-design.md) |
| [2. Access Control](./testing-items/2-access-control.md) |
| [3. Error Handling and Logging](./testing-items/3-error-handling-and-logging.md) |
| [4. Business Logic](./testing-items/4-business-logic.md) |
| [5. Blockchain Data](./testing-items/5-blockchain-data.md) |
| [6. External Components](./testing-items/6-external-components.md) |
| [7. Arithmetic](./testing-items/7-arithmetic.md) |
| [8. Denial of Services](./testing-items/8-denial-of-services.md) |
| [9. Best Practices](./testing-items/9-best-practices.md) |

## Authors

The following people have contributed to the creation of this document:

- [ErbaZZ](https://github.com/ErbaZZ) (Weerawat Pawanawiwat - weerawat.p@inspex.co)
- [Rugsurely](https://github.com/Rugsurely) (Patipon Suwanbol - patipon.s@inspex.co)
- [DeStinE21](https://github.com/DeStinE21) (Natsasit Jirathammanuwat - natsasit.j@inspex.co)
- [Jusmistic](https://github.com/Jusmistic) (Puttimate Thammasaeng - puttimate.t@inspex.co)
- [x0r4w](https://github.com/x0r4w) (Sorawish Laovakul - sorawish.l@inspex.co)
- [ICQCQ](https://github.com/ICQCQ) (Darunphop Pengkumta - darunphop.p@inspex.co)
- [jokopoppo](https://github.com/jokopoppo) (Wachirawit Kanpanluk - wachirawit.k@inspex.co)

## Acknowledgement

We would like to thank the authors of the these amazing resources that were used in the creation of this document:

- [Ethereum Smart Contract Security Best Practices](https://consensys.github.io/smart-contract-best-practices/)
- [Smart Contract Weakness Classification and Test Cases](https://swcregistry.io/)
- [Solidity Patterns](https://fravoll.github.io/solidity-patterns/)
- [OpenZeppelin Contracts](https://github.com/OpenZeppelin/openzeppelin-contracts)
- [Solidity Documentation](https://docs.soliditylang.org/en/v0.8.13/)
- [Solidity by Example](https://solidity-by-example.org/)
- [Slither, the Solidity source analyzer](https://github.com/crytic/slither)
- [Smart Contract Security Verification Standard](https://github.com/securing/SCSVS)

## Disclaimers

This document is for educational purposes. The smart contract source code in this document contains vulnerabilities, and does not guarantee the safety of the smart contract when used.

**DO NOT USE any of the source code in this document on production.**

## License

This work is licensed under a
[Creative Commons Attribution-ShareAlike 4.0 International License][cc-by-sa].

[cc-by-sa]: http://creativecommons.org/licenses/by-sa/4.0/
[cc-by-sa-image]: https://licensebuttons.net/l/by-sa/4.0/88x31.png
[cc-by-sa-shield]: https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg