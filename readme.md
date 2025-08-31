# Web3 Vulnerabilities Repository

A comprehensive collection of clustered smart contract vulnerabilities discovered through security audits, organized by severity and frequency of occurrence.

## 📊 Overview

This repository contains a systematically organized database of Web3/smart contract vulnerabilities discovered across numerous security audits. Each vulnerability has been automatically clustered using machine learning techniques to group similar issues, providing insights into the most common patterns of smart contract bugs.

## 🔍 What's Inside

### Vulnerability Clusters
- **29k+ unique vulnerabilities** across **461 clusters**
- Ranked by frequency of occurrence
- Detailed examples with code snippets and impact analysis
- Automated labeling with human verification

### Structure
```
📁 
├── 📄 readme.md                 # This file
├── 📁 cluster_pages/
│   ├── 📄 index.md                  # Complete cluster index
│   ├── 📄 cluster_-1015.md 
│   ├── 📄 cluster_-1030.md
│   └──  📄 cluster_-n.md
```

## 🎯 Top Vulnerability Categories

| Rank | Count | Category | Severity |
|------|-------|----------|----------|
| 1 | 1879 | Use of unchecked arithmetic operations bypasses overflow/underflow checks, introducing risks of incorrect state transitions, exploitable integer overflows, and compromised arithmetic integrity in critical financial calculations. | [View](cluster_pages/cluster_-1015.md) |
| 2 | 1227 | Failure to validate account ownership or token identity leads to unauthorized access, mint manipulation, or invalid state transitions, enabling attackers to bypass authorization and exploit token or account mismatches. | [View](cluster_pages/cluster_-1030.md) |
| 3 | 1085 | Inconsistent reward accounting and improper stake validation lead to incorrect or unfair reward distribution, enabling economic manipulation, loss of trust, and denial-of-service through flawed data tracking or unchecked access. | [View](cluster_pages/cluster_-1395.md) |
| 4 | 1068 | Failure to validate or remove token state entries enables malicious actors to manipulate contract state, leading to denial of service, incorrect rewards, or unauthorized transfers through inadequate input checks or missing state updates. | [View](cluster_pages/cluster_-1041.md) |
| 5 | 1049 | Precision loss and logic flaws in reward calculations due to improper handling of fixed-point arithmetic, incorrect loop bounds, and inconsistent state updates, leading to systematic underpayment, denial of service, and wealth leakage. | [View](cluster_pages/cluster_-1005.md) |

*[View complete ranking →](clustered_pages/index.md)*

## 📖 How to Use This Repository

This is strict

### For Security Researchers
- Browse clusters by frequency to understand common vulnerability patterns
- Use examples as reference for audit findings
- Use for educational materials

### For Developers
- Review top clusters before deploying contracts
- Use as a checklist during code review
- Learn from real-world examples with PoCs

## 🔬 Cluster Example

Here's what you'll find in each cluster page:

```markdown
# Cluster -1450
**Rank:** #166  
**Count:** 77  

## Label
Inaccurate state updates due to improper edge case handling...

## Examples
### Example 1 - Rounding Errors in Vault Withdrawals
- **Impact:** Value leakage from protocol
- **Code Analysis:** Detailed breakdown with vulnerable snippets
- **Proof of Concept:** Working exploit demonstration
- **Mitigation:** Specific fix recommendations
```

## 🛡️ Common Vulnerability Patterns

### 1. **Arithmetic Issues** (1,879 cases)
- Integer overflow/underflow
- Unchecked arithmetic operations
- Precision loss in calculations

### 2. **Access Control** (1,227 cases)
- Missing authorization checks
- Role-based permission bypass
- Token ownership validation failures

### 3. **Economic Logic** (1,085 cases)
- Reward calculation errors
- Unfair distribution mechanisms
- Stake validation issues

### 4. **State Management** (892 cases)
- Reentrancy vulnerabilities
- State synchronization issues
- Cross-function state corruption

## 📈 Data Sources

Data is scrapped from publicly available sources such as Code4rena, Cantina, Sherlock and other publicly available reports.

## 🔧 Methodology

### Clustering Process
1. **Data Collection**: Automated scraping of audit reports
2. **Text Processing**: NLP preprocessing of vulnerability descriptions
3. **Similarity Analysis**: Machine learning clustering algorithms
4. **Human Review**: Expert validation of cluster assignments
5. **Automated Labeling**: AI-generated labels with human oversight

## 📄 License

This project is licensed under the MIT License
