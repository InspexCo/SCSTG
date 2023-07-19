# Smart Contract Security Testing Guide

[![CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-sa/4.0/)

Smart Contract Security Testing Guide (SCSTG) is a risk-based guide for smart contract security professionals and developers to use as a reference in the security testing of smart contracts. It describes the characteristics and processes for verifying smart contract security issues in different categories, together with examples of vulnerable contracts or functions, and solutions to resolving the risks from their root causes or mitigating their risks.

## Testing Categories

The risks are categorized into 9 categories. Each page includes the description of the risks, examples, and solution or mitigations for those risks.

The risks are categorized in 9 categories as follows:

1. [**Testing Arithmetic Operation and Conversion**](testing-items/1-architecture-and-design.md) - Mathematical operations on different programming languages and platforms may work differently. The arithmetic operations done in the smart contract should be able to safely handle the whole range of possible values.
2. [**Testing Contract Compiling**](testing-items/2-access-control.md) - Smart contract can be implemented in various ways, depending on each developer’s style. However, complying with the best practices can improve the code quality of the smart contract, making it cleaner, more readable, or more efficient.
3. [**Testing External Interaction**](testing-items/3-error-handling-and-logging.md) - Smart contracts can be interconnected through the inheritance of the previously developed smart contracts or the calling of functions from other contracts. Usage of insecure external components can cause undesirable or harmful effects if not done properly.
4. [**Testing Privilege Function**](testing-items/4-business-logic.md) - Smart contracts often include functions that require certain privileges or access levels to be executed. Testing privilege functions involves verifying that only authorized users or roles can invoke these functions, while unauthorized users are prevented from accessing them. This ensures that sensitive operations and data within the contract are adequately protected from unauthorized manipulation.
5. [**Testing Control Flow**](testing-items/5-blockchain-data.md) - Control flow refers to the order in which instructions are executed within a smart contract. Testing control flow involves verifying that the contract behaves as expected under different scenarios and conditions. This includes testing conditional statements (if-else), loops, function calls, and other control structures.
6. [**Testing Access Control**](testing-items/6-external-components.md) -  Access control is the imposing of policy by preventing users from acting beyond the scope of their authorized permissions. Improper access control can lead to unauthorized information disclosure, data manipulation or loss, or performing of business functions outside the user's capability.
7. [**Testing Randomness**](testing-items/7-arithmetic.md) - Randomness is often a crucial aspect of smart contracts, particularly in applications such as gambling, games, or random selection processes. Testing randomness involves assessing the reliability and fairness of the random number generation (RNG) mechanisms employed in the contract. It includes checking the source of randomness, evaluating the distribution of generated numbers, and verifying that the RNG cannot be manipulated or predicted by any party.
8. [**Testing Loop Operation**](testing-items/8-denial-of-services.md) - Smart contracts often involve loop structures, such as for loops and while loops, to iterate through data or execute repetitive tasks. Testing loop operation focuses on verifying the correctness and efficiency of these loops. It includes testing the initialization, condition evaluation, loop body execution, and termination conditions. By testing loop operations, developers can ensure that the contract handles iterations correctly, avoids infinite loops, and performs optimally without excessive gas consumption or potential vulnerabilities.
9. [**Testing Contract Upgradability**](testing-items/9-best-practices.md) - Contract upgradability is a critical aspect of smart contract design, allowing for future enhancements and bug fixes without requiring a complete redeployment. Testing contract upgradability involves verifying that the contract has been designed to accommodate upgrades effectively. It includes testing the mechanisms for separating data and logic, ensuring backward compatibility, preserving the contract's state, and securely transitioning to a new version.

## Authors

The following people have contributed to the creation of this document:

* [ErbaZZ](https://github.com/ErbaZZ) (Weerawat Pawanawiwat - weerawat.p@inspex.co)
* [Rugsurely](https://github.com/Rugsurely) (Patipon Suwanbol - patipon.s@inspex.co)
* [DeStinE21](https://github.com/DeStinE21) (Natsasit Jirathammanuwat - natsasit.j@inspex.co)
* [Jusmistic](https://github.com/Jusmistic) (Puttimet Thammasaeng - puttimet.t@inspex.co)
* [x0r4w](https://github.com/x0r4w) (Sorawish Laovakul - sorawish.l@inspex.co)
* [ICQCQ](https://github.com/ICQCQ) (Darunphop Pengkumta - darunphop.p@inspex.co)
* [jokopoppo](https://github.com/jokopoppo) (Wachirawit Kanpanluk - wachirawit.k@inspex.co)

## Acknowledgement

We would like to thank the authors of the these amazing resources that were used in the creation of this document:

* [Ethereum Smart Contract Security Best Practices](https://consensys.github.io/smart-contract-best-practices/)
* [Smart Contract Weakness Classification and Test Cases](https://swcregistry.io/)
* [Solidity Patterns](https://fravoll.github.io/solidity-patterns/)
* [OpenZeppelin Contracts](https://github.com/OpenZeppelin/openzeppelin-contracts)
* [Solidity Documentation](https://docs.soliditylang.org/en/v0.8.13/)
* [Solidity by Example](https://solidity-by-example.org/)
* [Slither, the Solidity source analyzer](https://github.com/crytic/slither)
* [Smart Contract Security Verification Standard](https://github.com/securing/SCSVS)

## Disclaimers

This document is for educational purposes. The smart contract source code in this document contains vulnerabilities, and does not guarantee the safety of the smart contract when used.

**DO NOT USE any of the source code in this document on production.**

## License

This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).
