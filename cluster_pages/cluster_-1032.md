# Cluster -1032

**Rank:** #244  
**Count:** 38  

## Label
Function signature collisions and improper function selector handling lead to unintended function routing, access control failures, or logic errors due to identical 4-byte selectors across contracts or missing interface differentiation.

## Cluster Information
- **Total Findings:** 38

## Examples

### Example 1

**Auto Label:** Failure to properly include or validate function selectors in diamond contracts, leading to inaccessible functionality, incorrect interface detection, and potential unauthorized state manipulation or protocol errors.  

**Original Text Preview:**

A test-only `ViewFacet` facet is being included directly in the production code, which could potentially be used by external contracts as a source of truth. For example, the `ViewFacet.getVertex()` function returns only the first possible vertex for test-only usage.

The `ViewFacet` should not be included in production. It should only be added in tests to the SimplexDiamond `production` diamond cut by using the `diamondCut` function and adding it for testing purposes only.

Consider removing the `ViewFacet` facet from the SimplexDiamond initial diamond cut.

---
### Example 2

**Auto Label:** Function selector collisions enable unauthorized function execution across contracts due to shared 4-byte selectors, allowing attackers to invoke malicious logic through selector-based access controls or proxy invocations.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `SimplexFacet` contract is missing the `setAdjustor` selector from the `simplexSelectors` array. This means that the `setAdjustor` function cannot be called.

## Recommendations

```diff
-       bytes4[] memory simplexSelectors = new bytes4[](9);
+       bytes4[] memory simplexSelectors = new bytes4[](10);
        simplexSelectors[0] = SimplexFacet.getVertexId.selector;
        simplexSelectors[1] = SimplexFacet.addVertex.selector;
        simplexSelectors[2] = SimplexFacet.getTokens.selector;
        simplexSelectors[3] = SimplexFacet.getIndexes.selector;
        simplexSelectors[4] = SimplexFacet.numVertices.selector;
        simplexSelectors[5] = SimplexFacet.withdrawFees.selector;
        simplexSelectors[6] = SimplexFacet.setDefaultEdge.selector;
        simplexSelectors[7] = SimplexFacet.setName.selector;
        simplexSelectors[8] = SimplexFacet.getName.selector;
+       simplexSelectors[9] = SimplexFacet.setAdjustor.selector;
```

---
### Example 3

**Auto Label:** Failure to properly include or validate function selectors in diamond contracts, leading to inaccessible functionality, incorrect interface detection, and potential unauthorized state manipulation or protocol errors.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `SimplexDiamond` contract is intended to support multiple interfaces, including `IERC165`, `IDiamondCut`, `IDiamondLoupe`, and `IERC173`. While `SimplexDiamond` correctly adds these interfaces to `ds.supportedInterfaces`, it **fails to include the `supportsInterface` function from `DiamondLoupeFacet` as a selector**. This omission **prevents the contract from properly supporting interfaces**, making it **incompatible with standard interface detection mechanisms**.

## Recommendations

Ensure `supportsInterface` is included in the `DiamondLoupeFacet` selectors:

```diff
        {
-            bytes4[] memory loupeFacetSelectors = new bytes4[](4);
+            bytes4[] memory loupeFacetSelectors = new bytes4[](5);
            loupeFacetSelectors[0] = DiamondLoupeFacet.facets.selector;
            loupeFacetSelectors[1] = DiamondLoupeFacet.facetFunctionSelectors.selector;
            loupeFacetSelectors[2] = DiamondLoupeFacet.facetAddresses.selector;
            loupeFacetSelectors[3] = DiamondLoupeFacet.facetAddress.selector;
+            loupeFacetSelectors[4] = DiamondLoupeFacet.supportsInterface.selector;
            cuts[1] = FacetCut({
                facetAddress: address(new DiamondLoupeFacet()),
                action: FacetCutAction.Add,
                functionSelectors: loupeFacetSelectors
            });
```

---
