## 1. Generating OD Demand at the Postcode Level

In this step, we generate the origin-destination (OD) demand at the postcode level by creating a postcode travel time matrix.

#### Inputs:
- **Travel Time by Travel Zones**: A dataset with columns:
  - `From` (origin travel zone)
  - `To` (destination travel zone)
  - `Minutes` (travel time in minutes)
- **Concordance Table**: A mapping table with columns:
  - `Travel Zone`
  - `Name`
  - `Postcode`

#### Output:
- A table of travel times between postcodes.

#### Process:
- The travel times (originally defined between travel zones) are mapped to postcodes using the concordance table.
- For each row of travel time data, the function selects a representative travel time based on the following logic:
  - **Fewer than 10 values**: It chooses the highest travel time to be conservative.
  - **10 or more values**: It selects a value below the maximum, adjusted by the data’s spread (capped at 50), to avoid overreacting to extreme outliers.

### Code Implementation
Here’s the Python function that processes the travel time data:

```python
def getTravelTime(row):
    travelTimeArray = list(map(float, row['Travel Time'].split(';')))
    if len(travelTimeArray) < 10:
        percentileAdjustment = 0
    else:
        percentileAdjustment = min(2 * max(travelTimeArray), 50)
    return np.percentile(travelTimeArray, 100 - percentileAdjustment)
``` 
## 2. Generating Freight Link Flow Proportion Graphs

This process recalculates the output link share for each gateway entry using logistic chain flow data, which specifies the proportion of freight exiting each link. By aggregating shares across routes and between linked nodes, it enables the calculation of Twenty-foot Equivalent Units (TEUs) from the total volume at each gateway. This step serves as a critical bridge between high-level demand (total freight per dock), mid-level demand (flows between docks), and eventual postcode-level demand.

### Inputs
- **Gateway Volume**: 
  - Contains the total freight volume crossing each gateway.
  - Unique entries identified by: `Gateway`, `Sector`, `Import / Export`, `Full / Empty`.
- **Logistic Chain Flows**: 
  - Details the chain of freight links with independent output shares for each link.

### Outputs
- **Logistic Chain Flows (with Output Link Share)**: 
  - An updated chain flow table including calculated output link shares between gateways.

### Process
1. **Traversal and Matching**:
   - Each gateway from the `gatewayVolumes` table is traversed and matched to `logisticChainFlows` using the columns: `['Gateway', 'Sector', 'Import / Export', 'Full / Empty']`.
   - A multi-column matching function filters the logistic chain table.

   ```python
   def matchStrings(primaryString, secondaryString):
       isMatch = False
       secondaryStringList = secondaryString.split(';')
       for i in range(len(secondaryStringList)):
           isMatch = isMatch | fnmatch.fnmatch(primaryString, secondaryStringList[i])
       return isMatch

   def matchCols(primaryRow, secondaryRow, primaryColsList, secondaryColsList):
       rowMatch = True
       for j in range(len(primaryColsList)):
           rowMatch = rowMatch & \
                   matchStrings(primaryRow[primaryColsList[j]], \
                                secondaryRow[secondaryColsList[j]])
       return rowMatch
    ```
2. **Example Run**

Here’s how the process works using `Swanson Dock West` (Import, Full) as an example:

### Input Data
- **From `gatewayVolumes`:**  
  Gateway: Swanson Dock West, Sector: I&MC, Import / Export: Import, Full / Empty: Full, Y2056: 4209.470819953
- **From `logisticChainFlows` (subset):**  
  - From: Swanson Dock West, To: PRS Terminals WIFT / BIFT, Y2056: 0.121441926471351  
  - From: Swanson Dock West, To: Importers, Y2056: 0.176977290524667  
  - From: Swanson Dock West, To: Transport Depots, Y2056: 0.617909481540465  

### Steps
- **Initialization:**  
  Entry point: Swanson Dock West, initial Output Link Share = 1.0 (100% of 4209.470819953 TEUs).
- **Iteration 1: From Swanson Dock West**  
  Splits volume based on Y2056 shares:  
  - PRS Terminals WIFT / BIFT: `1.0 * 0.121441926471351 = 0.1214`  
  - Importers: `1.0 * 0.176977290524667 = 0.1770`  
  - Transport Depots: `1.0 * 0.617909481540465 = 0.6179`  
- **Iteration 2: Propagation**  
  - From PRS Terminals WIFT / BIFT (0.1214):  
    - Importers: `0.1214 * 0.7 = 0.0850`  
  - From Transport Depots (0.6179):  
    - Importers: `0.6179 * 1.0 = 0.6179`  
  - From Importers (0.1770 direct):  
    - Container Parks: `0.1770 * 0.42218675179569 = 0.0747`  
  - Importers total share: `0.1770 + 0.0850 + 0.6179 = 0.8799`  
- **Iteration 3: From Importers**  
  - Container Parks: `0.8799 * 0.42218675179569 = 0.3715`  
  - Total to Container Parks: `0.0747 + 0.3715 = 0.4462`  
- **Termination:**  
  No further outgoing flows from Container Parks in this subset, so the process stops.

### Resulting Flow Graph
```markdown
Swanson Dock West (Entry, 1.0)
  ├──> PRS Terminals WIFT / BIFT (0.1214)
  │     └──> Importers (0.0850)
  │           └──> Container Parks (0.3715 cumulative)
  ├──> Importers (0.1770 direct)
  │     └──> Container Parks (0.0747 initial, 0.4462 total)
  └──> Transport Depots (0.6179)
        └──> Importers (0.6179)
              └──> Container Parks (included in cumulative total)
```
**Cumulative Shares**:
  - Importers: 0.8799 (0.1770 + 0.0850 + 0.6179)
  - Container Parks: 0.4462 (0.0747 + 0.0359 + 0.2608)

## 3. Generating postcode volume
The input/output flow between each postcode is calculated  in this step. 
### Inputs
- **Gateway Volume**: 
  - Contains the total freight volume crossing each gateway.
  - Unique entries identified by: `Gateway`, `Sector`, `Import / Export`, `Full / Empty`.
- **Logistic Chain Flows (with output link share)**: 
  - Details the chain of freight links with independent output shares for each link.
- **Postcode shares**:
    - Maps freight chain links to specific geographic locations (postcodes) and provides associated freight volumes or shares.

### Outputs
- **Input/Output volume for each Postcode**: 
 
### Process
Example Run: Swanson Dock West (Import, Full)

### Initial Setup
- **Gateway**: `Swanson Dock West, I&MC, Import, Full, IMEX, Y2056: 4209.470819953`
- **Matched Flows**: Filtered from `chainFlowVolumes` (e.g., to `PRS Terminals WIFT / BIFT`, `Importers`, `Transport Depots`).
- **Matched Shares**: Filtered from `postcodeShares` (e.g., `Swanson Dock West` to `3003`, `Importers` to various postcodes).

### Process
Filter the postcode and link chain volume with entry:

**Gateway Row**: Swanson Dock West, I&MC, Import, Full, IMEX, Y2056: 4209.470819953

Then merged all of them we have this:


Gateway            | Sector | Import / Export | Full / Empty | Industry Class | To Freight Area            | To Postcode | To Share           | From               | To                      | From State | To State | Run Type | Output Link Share
-------------------|--------|-----------------|--------------|----------------|----------------------------|-------------|---------------------|--------------------|-------------------------|------------|----------|----------|-------------------
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | WIFT West Rail Terminal    | 3029        | 0.53756397021675   | Swanson Dock West  | PRS Terminals WIFT / BIFT | Full       | Full     | RL       | 0.121441926471
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | WIFT East Rail Terminal    | 3029        | 0.276944268216206  | Swanson Dock West  | PRS Terminals WIFT / BIFT | Full       | Full     | RL       | 0.121441926471
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3003        | 201090.450585583   | Swanson Dock West  | Importers               | Full       | Empty    | CR       | 0.176977290525
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3011        | 276314.939418594   | Swanson Dock West  | Importers               | Full       | Empty    | CR       | 0.176977290525
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3175        | 0.102144194376892  | Swanson Dock West  | Transport Depots        | Full       | Full     | BR       | 0.61790948154
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3003        | 0.360811314652934  | Swanson Dock West  | Transport Depots        | Full       | Full     | BR       | 0.61790948154
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3003        | 201090.450585583   | PRS Terminals WIFT / BIFT | Importers     | Full       | Empty    | CR       | 0.0850093485299
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3003        | 201090.450585583   | Transport Depots    | Importers               | Full       | Empty    | CR       | 0.61790948154
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3003        | 0.0874510024305604 | Importers          | Container Parks         | Empty      | Complete | CR       | 0.403272908557
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3011        | 0.111169455488899  | Importers          | Container Parks         | Empty      | Complete | CR       | 0.403272908557

**Notes**: To Share for Importers is in absolute TEUs (e.g., 201090.45), while others are proportions (0–1).

```python
shareVolumes['Input Combined Shares'] = shareVolumes['Output Link Share'] * shareVolumes['To Share']
```
To                      | To Postcode | To Share           | Output Link Share  | Input Combined Shares
------------------------|-------------|--------------------|--------------------|----------------------
PRS Terminals WIFT / BIFT | 3029       | 0.53756397021675   | 0.121441926471     | 0.065282685
PRS Terminals WIFT / BIFT | 3029       | 0.276944268216206  | 0.121441926471     | 0.033627856
Importers               | 3003        | 201090.450585583   | 0.176977290525     | 35589.779224
Importers               | 3011        | 276314.939418594   | 0.176977290525     | 48896.379341
Transport Depots        | 3175        | 0.102144194376892  | 0.61790948154      | 0.063119852
Transport Depots        | 3003        | 0.360811314652934  | 0.61790948154      | 0.222878773
Importers               | 3003        | 201090.450585583   | 0.0850093485299    | 17092.954640
Importers               | 3003        | 201090.450585583   | 0.61790948154      | 124252.614120
Container Parks         | 3003        | 0.0874510024305604 | 0.403272908557     | 0.035266141
Container Parks         | 3011        | 0.111169455488899  | 0.403272908557     | 0.044837773

```python
currentScale = shareVolumes.groupby(['To', 'To State']).agg({'Input Combined Shares': 'sum'}).reset_index().rename(columns={'Input Combined Shares': 'Current Scale'})
goalScale = matchedFlows.groupby(['To', 'To State']).agg({'Output Link Share': 'sum'}).reset_index().rename(columns={'Output Link Share': 'Goal Scale'})
shareVolumes = shareVolumes.merge(currentScale).merge(goalScale)
shareVolumes['Input Combined Shares'] = shareVolumes['Input Combined Shares'] / shareVolumes['Current Scale'] * shareVolumes['Goal Scale']
```
To                      | To Postcode | To Share           | Output Link Share  | Input Combined Shares
------------------------|-------------|--------------------|--------------------|----------------------
PRS Terminals WIFT / BIFT | 3029       | 0.53756397021675   | 0.121441926471     | 0.080136
PRS Terminals WIFT / BIFT | 3029       | 0.276944268216206  | 0.121441926471     | 0.041306
Importers               | 3003        | 201090.450585583   | 0.176977290525     | 0.138672
Importers               | 3011        | 276314.939418594   | 0.176977290525     | 0.190477
Transport Depots        | 3175        | 0.102144194376892  | 0.61790948154      | 0.136347
Transport Depots        | 3003        | 0.360811314652934  | 0.61790948154      | 0.481562
Importers               | 3003        | 201090.450585583   | 0.0850093485299    | 0.066629
Importers               | 3003        | 201090.450585583   | 0.61790948154      | 0.484118
Container Parks         | 3003        | 0.0874510024305604 | 0.403272908557     | 0.177548
Container Parks         | 3011        | 0.111169455488899  | 0.403272908557     | 0.225725


Verification
PRS Terminals WIFT / BIFT: 0.080136 + 0.041306 = 0.121442 (matches 0.121441926471 within rounding).

Importers: 0.138672 + 0.190477 + 0.066629 + 0.484118 = 0.879896 (matches goal).

Transport Depots: 0.136347 + 0.481562 = 0.617909 (matches goal).

Container Parks: 0.177548 + 0.225725 = 0.403273 (matches goal).

```python
# Add gateway entry
entryPoint = matchedFlows['Volume Entry'].iloc[0]  # 'Swanson Dock West'
gatewayRows = matchedShares[matchedShares['Chain Link'] == entryPoint]\
            .rename(columns={'Chain Link': 'To', 'Postcode': 'To Postcode', 'Freight Area': 'To Freight Area', 'Y2056': 'Output Combined Shares'})
for i in range(len(gatewayMatchColNames)):
    gatewayRows[gatewayMatchColNames[i]] = gatewayRow[gatewayMatchColNames[i]]
gatewayRows['Industry Class'] = gatewayRow['Industry Class']
gatewayRows['To State'] = matchedFlows[matchedFlows['From'] == entryPoint].iloc[0].loc['From State']
shareVolumes = pd.concat([shareVolumes, gatewayRows], ignore_index=True)

# Assign Output Combined Shares
shareVolumes.loc[(shareVolumes['To State'].isin(shareVolumes['From State'])), 'Output Combined Shares'] = shareVolumes['Input Combined Shares']
```
Gateway            | Sector | Import / Export | Full / Empty | Industry Class | To Freight Area            | To Postcode | To Share           | From               | To                      | From State | To State | Run Type | Output Link Share  | Input Combined Shares | Output Combined Shares
-------------------|--------|-----------------|--------------|----------------|----------------------------|-------------|--------------------|--------------------|-------------------------|------------|----------|----------|--------------------|-----------------------|------------------------
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | WIFT West Rail Terminal    | 3029        | 0.53756397021675   | Swanson Dock West  | PRS Terminals WIFT / BIFT | Full       | Full     | RL       | 0.121441926471     | 0.080136             | 0.080136
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | WIFT East Rail Terminal    | 3029        | 0.276944268216206  | Swanson Dock West  | PRS Terminals WIFT / BIFT | Full       | Full     | RL       | 0.121441926471     | 0.041306             | 0.041306
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3003        | 201090.450585583   | Swanson Dock West  | Importers               | Full       | Empty    | CR       | 0.176977290525     | 0.138672             | 0.138672
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3011        | 276314.939418594   | Swanson Dock West  | Importers               | Full       | Empty    | CR       | 0.176977290525     | 0.190477             | 0.190477
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3175        | 0.102144194376892  | Swanson Dock West  | Transport Depots        | Full       | Full     | BR       | 0.61790948154      | 0.136347             | 0.136347
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3003        | 0.360811314652934  | Swanson Dock West  | Transport Depots        | Full       | Full     | BR       | 0.61790948154      | 0.481562             | 0.481562
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3003        | 201090.450585583   | PRS Terminals WIFT / BIFT | Importers        | Full       | Empty    | CR       | 0.0850093485299    | 0.066629             | 0.066629
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3003        | 201090.450585583   | Transport Depots    | Importers               | Full       | Empty    | CR       | 0.61790948154      | 0.484118             | 0.484118
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3003        | 0.0874510024305604 | Importers          | Container Parks         | Empty      | Complete | CR       | 0.403272908557     | 0.177548             | NaN
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           |                            | 3011        | 0.111169455488899  | Importers          | Container Parks         | Empty      | Complete | CR       | 0.403272908557     | 0.225725             | NaN
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | Port of Melbourne - Swanson Dock West | 3003 | 1                  | NaN                | Swanson Dock West       | NaN        | Full     | NaN      | NaN                | NaN                  | 1.0

```python
shareVolumes = shareVolumes.groupby(gatewayMatchColNames + ['Industry Class', 'To', 'To Postcode', 'To State'])\
                           .agg({'Input Combined Shares': 'sum', 'Output Combined Shares': 'sum'})\
                           .reset_index()
shareVolumes['Input Volume'] = shareVolumes['Input Combined Shares'] * gatewayRow[yearColName]
shareVolumes['Output Volume'] = shareVolumes['Output Combined Shares'] * gatewayRow[yearColName]
shareVolumes = shareVolumes[gatewayMatchColNames + ['Industry Class', 'To', 'To Postcode', 'To State', 'Input Volume', 'Output Volume']]
```
Gateway            | Sector | Import / Export | Full / Empty | Industry Class | To                      | To Postcode | To State | Input Volume | Output Volume
-------------------|--------|-----------------|--------------|----------------|-------------------------|-------------|----------|--------------|---------------
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | Swanson Dock West       | 3003        | Full     | 0.00         | 4209.47
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | PRS Terminals WIFT / BIFT | 3029      | Full     | 511.14       | 511.14
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | Importers               | 3003        | Empty    | 2901.88      | 2901.88
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | Importers               | 3011        | Empty    | 801.59       | 801.59
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | Transport Depots        | 3175        | Full     | 573.95       | 573.95
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | Transport Depots        | 3003        | Full     | 2026.98      | 2026.98
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | Container Parks         | 3003        | Complete | 747.44       | 0.00
Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | Container Parks         | 3011        | Complete | 950.00       | 0.00

## 4. Generating postcode travel time for chainlinks
This script will map travel time which was previously generated (from/to colume) with the postcode-chainlink map to get the postcode chain link travel time.
| Gateway            | Sector | Import / Export | Full / Empty | Industry Class | Run Type | From Postcode | From              | From State | To Postcode | To                       | To State | Travel Time |
|--------------------|--------|-----------------|--------------|----------------|----------|---------------|-------------------|------------|-------------|--------------------------|----------|-------------|
| Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | RL       | 3003          | Swanson Dock West | Full       | 3029        | PRS Terminals WIFT / BIFT | Full     | 27.543957   |
| Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | RL       | 3003          | Swanson Dock West | Full       | 3029        | PRS Terminals WIFT / BIFT | Full     | 27.543957   |
| Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | RL       | 3003          | Swanson Dock West | Full       | 3753        | PRS Terminals WIFT / BIFT | Full     | 44.769091   |
| Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | RL       | 3003          | Swanson Dock West | Full       | 3753        | PRS Terminals WIFT / BIFT | Full     | 44.769091   |
| Swanson Dock West  | I&MC   | Import          | Full         | IMEX           | RL       | 3003          | Swanson Dock West | Full       | 3018        | PRS Terminals Others      | Full     | 18.705176   |
| ...                | ...    | ...             | ...          | ...            | ...      | ...           | ...               | ...        | ...         | ...                      | ...      | ...         |


## 5. Optimisation

### Notation

#### Sets:
- \( I \): Set of input nodes (from `inputMarginals`), indexed by \( i \) (e.g., `To Postcode`, `To`, `To State` combinations with non-zero `Input Volume`).
- \( J \): Set of output nodes (from `outputMarginals`), indexed by \( j \) (e.g., `To Postcode`, `To`, `To State` combinations with non-zero `Output Volume`).
- \( K \): Set of possible routes (from `matchedTimes`), indexed by \( k \) (e.g., `From Postcode` to `To Postcode` pairs).

#### Parameters:
- \( c_k \): Travel time for route \( k \) (from `Travel Time` in `matchedTimes`, default 240 if missing).
- \( w \): Spread weight penalty for auxiliary variables (denoted `spreadWeight` in code, assumed constant).
- \( s_i \): Supply (input volume) at input node \( i \) (from `Input Volume` in `inputMarginals`).
- \( d_j \): Demand (output volume) at output node \( j \) (from `Output Volume` in `outputMarginals`).

#### Decision Variables:
- \( x_k \): Freight volume (TEUs) transported along route \( k \), where \( x_k \geq 0 \).
- \( z_j \): Auxiliary variable for output node \( j \), representing slack or spread, where \( z_j \geq 0 \).

### Objective Function
Minimize the total travel time plus a penalty for auxiliary variables:

\[
\text{Minimize} \quad Z = \sum_{k \in K} c_k x_k + w \sum_{j \in J} z_j
\]

- \( \sum_{k \in K} c_k x_k \): Total travel time across all routes.
- \( w \sum_{j \in J} z_j \): Penalty term to control the spread of flows (auxiliary variables ensure feasibility).