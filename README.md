# Retirement_Strategy_Simulator
Comparing the Three Bucket Strategy to the more simple Total Portfolio Rebalance Strategy

---------------------------------------------
THREE BUCKET PORTFOLIO METHODOLOGY EXPLAINED
---------------------------------------------
Let's explore the rebalancing logic for the Three-Bucket Strategy. This one has a bit more "if this, then that" personality compared to the Total Return strategy, which makes it interesting!

Imagine you're a very organized planner for your retirement funds, and you've divided your money into three distinct buckets:

Bucket 1 (B1): Cash Jar. This is for your immediate living expenses, maybe the next 1-3 years. You want to keep this pretty full.

Bucket 2 (B2): Steady Savers Jar (Bonds). This is for medium-term stability, meant to refill the Cash Jar. It's less risky than stocks.

Bucket 3 (B3): Growth Engine Jar (Stocks). This is your long-term growth engine. It can be bouncy (volatile), but hopefully grows the most over time.

The rebalancing rules are all about keeping these jars at their desired levels and deciding when and how to move money between them.

Here's the core rebalancing section from simulateThreeBucketStrategy_Detailed. This happens after you've taken money out for the year's expenses and after each bucket's investments have had their yearly growth (or shrinkage).

    // SCENE: End of the year. We've paid our bills (withdrawals are done).
    // Our Cash Jar (B1), Steady Savers Jar (B2), and Growth Engine Jar (B3)
    // have all seen some action from market returns.
    // `b3MarketReturnThisYear` tells us if our Growth Engine Jar (B3 - Stocks) had a good year (positive return) or a bad year (negative return).
    // This is a VERY important piece of information for our decisions!
    
    // First, let's figure out the ideal dollar amount for our Cash Jar (B1) and Steady Savers Jar (B2).
    // `yearWithdrawal` is the amount of money we plan to spend *next* year (it includes inflation).
    // `params.bucket1Years` is how many years of spending we want in our Cash Jar (e.g., 3 years).
    const b1TargetDollar = params.bucket1Years * yearWithdrawal;
    // `params.bucket2YearsBonds` is how many years of spending we want in our Steady Savers Jar (e.g., 5 years).
    const b2TargetDollar = params.bucket2YearsBonds * yearWithdrawal;
    
    // Let's make some notes in our logbook for this year's rebalancing.
    // We'll record the target amounts and any money moved.
    logEntry.b1TargetAmount = b1TargetDollar;
    logEntry.b2TargetAmount = b2TargetDollar;
    logEntry.b1RefillAmount = 0; // How much we added to B1
    logEntry.b1RefillSource = 'N/A'; // Where did the refill for B1 come from?
    logEntry.b2TransferToB1 = 0; // How much B2 gave to B1
    logEntry.b3TransferToB1 = 0; // How much B3 gave to B1 (spoiler: this strategy avoids this if B3 is down)
    logEntry.b2RebalanceTransfer = 0; // Net change in B2 from rebalancing (not counting B1 refill)
    logEntry.b3RebalanceTransfer = 0; // Net change in B3 from rebalancing

// --- THE BIG DECISION POINT: How did the Stock Market (Bucket 3) do this year? ---

    if (b3MarketReturnThisYear >= 0) {
        // ----- CONDITION 1: STOCKS (B3) WERE UP or FLAT! 🎉 -----
        // This is good news! We feel more comfortable moving money around.
        // We're going to do a "Holistic Re-allocation" - basically, re-sort all our money
        // across the three buckets to get them to their ideal levels.

    logEntry.reallocStrategy = "Market Up/Flat: Holistic"; // Note our strategy

    // Remember how much was in each bucket *before* this big re-sort.
    let b1Old = bucket1Balance;
    let b2Old = bucket2Balance;
    let b3Old = bucket3Balance;
    let totalPortfolioAfterGrowth = bucket1Balance + bucket2Balance + bucket3Balance; // All our money right now

    // Step 1: Fill the Cash Jar (B1) to its target.
    // We take money from the total pot to fill B1, but we can't put more in B1 than our total money.
    bucket1Balance = Math.min(totalPortfolioAfterGrowth, b1TargetDollar);

    // Step 2: See how much money is left after filling B1.
    let remainingForB2B3 = totalPortfolioAfterGrowth - bucket1Balance;

    // Step 3: Fill the Steady Savers Jar (B2) to its target using the remaining money.
    // Again, we can't put more in B2 than what's left.
    bucket2Balance = Math.min(remainingForB2B3, b2TargetDollar);

    // Step 4: Whatever money is left over after filling B1 and B2 goes into the Growth Engine Jar (B3).
    // This is where the excess profits from a good year often end up.
    bucket3Balance = remainingForB2B3 - bucket2Balance;

    // Let's log the net changes for each bucket.
    // For B1, this is how much it changed to meet its target.
    logEntry.b1RefillAmount = bucket1Balance - b1Old;
    if(logEntry.b1RefillAmount > 0) logEntry.b1RefillSource = "Portfolio Realloc";
    else if (logEntry.b1RefillAmount < 0) logEntry.b1RefillSource = "Portfolio Realloc (Surplus to B2/B3)"; // B1 might have been overfilled before!
    else logEntry.b1RefillSource = "N/A";

    // For B2 and B3, these are their net changes due to this big re-shuffle.
    logEntry.b2RebalanceTransfer = bucket2Balance - b2Old;
    logEntry.b3RebalanceTransfer = bucket3Balance - b3Old;

    } else {
        // ----- CONDITION 2: STOCKS (B3) WERE DOWN! 😟 -----
        // Uh oh, the stock market didn't do well. We need to be more careful
        // and try to avoid selling stocks (B3) when they are low if we can.
        // This is where the "bucket" idea really shines.

    logEntry.reallocStrategy = "Market Down: Conditional"; // Note our cautious strategy

    // Step 1 (for Market Down): Check if the Cash Jar (B1) needs a top-up.
    // `bucket1RefillTriggerLevel` is like a "low fuel" warning for B1 (e.g., if B1 has less than 1 year of expenses).
    const bucket1RefillTriggerLevel = params.bucket1RefillThresholdYears * yearWithdrawal;

    if (bucket1Balance < bucket1RefillTriggerLevel) {
        // B1 is below our "low fuel" warning! We need to refill it.
        // How much do we need to add to get B1 back to its full target (`b1TargetDollar`)?
        let amountToRefillB1 = Math.max(0, b1TargetDollar - bucket1Balance);

        if (amountToRefillB1 > 0) {
            // IMPORTANT RULE: If stocks (B3) are down, we ONLY use the Steady Savers Jar (B2 - Bonds) to refill B1.
            // We try NOT to sell from B3 (stocks) when they're down.
            const fromB2Refill = Math.min(amountToRefillB1, bucket2Balance); // Can't take more than B2 has.

            bucket1Balance += fromB2Refill; // Add to B1
            bucket2Balance -= fromB2Refill; // Subtract from B2

            // Log this transfer.
            if (fromB2Refill > 0) {
                logEntry.b1RefillAmount += fromB2Refill; // Record how much B1 received
                logEntry.b1RefillSource = 'B2 Only';     // Record where it came from
                logEntry.b2TransferToB1 = -fromB2Refill; // Record B2 gave this much away (negative for B2)
            }
        }
    }
    // If B1 was already above the trigger level, we don't force a refill from B2 if stocks are down.
    // We're trying to let B2 and B3 recover if possible.

    // Step 2 (for Market Down): Check the Steady Savers Jar (B2 - Bonds).
    // We do NOT sell stocks (B3) to top up B2 if B2 is low and stocks are down.
    // BUT, if B2 is OVERFLOWING (has more than its `b2TargetDollar`), we'll move the excess
    // to the Growth Engine Jar (B3 - Stocks). This is like buying stocks when they might be "on sale".
    let b2OldForRebalance = bucket2Balance; // What B2 had before this specific check
    let b3OldForRebalance = bucket3Balance; // What B3 had

    if (bucket2Balance > b2TargetDollar) {
        // B2 has too much money! Let's skim the extra.
        const excessB2 = bucket2Balance - b2TargetDollar;
        bucket2Balance -= excessB2;     // Take excess from B2
        bucket3Balance += excessB2;     // Put that excess into B3 (buy more stocks)
    }
    // If B2 is below target, it just stays below target for now. We don't take from B3.

    // Log the net changes for B2 and B3 from this specific "B2 overflow" rule.
    logEntry.b2RebalanceTransfer = bucket2Balance - b2OldForRebalance;
    logEntry.b3RebalanceTransfer = bucket3Balance - b3OldForRebalance;
    }
    
    // Make sure no bucket goes below zero (just a safety check).
    bucket1Balance = Math.max(0, bucket1Balance);
    bucket2Balance = Math.max(0, bucket2Balance);
    bucket3Balance = Math.max(0, bucket3Balance);
    
    // Finally, log the ending balances of each bucket after all these decisions.
    logEntry.b1End = bucket1Balance;
    logEntry.b2End = bucket2Balance;
    logEntry.b3End = bucket3Balance;


-----------------------------------------------
TOTAL PORTFOLIO REBALANCE METHODOLOGY EXPLAINED
-----------------------------------------------

Now, let's dive into the rebalancing magic for the Total Return strategy! It's actually quite straightforward once you see it step-by-step. Imagine you're a diligent portfolio manager for yourself, and at the end of each year, you want to make sure your investments are still in the right proportions.

Here's the part of the code in simulateTotalReturnStrategy_Detailed that handles this annual check-up and adjustment:
    
    // SCENE: This happens at the end of each simulated year,
    // AFTER your money has been withdrawn for living expenses,
    // AND AFTER your stocks and bonds have grown (or shrunk) based on market returns.
    
    // Initialize a couple of notes in our logbook for this year.
    // We'll write down how much stock and bond we *actually* bought or sold during rebalancing.
    // For now, we assume we haven't bought/sold anything for rebalancing yet, so they are zero.
    logEntry.rebalanceStockAmount = 0;
    logEntry.rebalanceBondAmount = 0;
    
    // First, a common-sense check: Is there any money left in the portfolio?
    // If the portfolio balance is zero (or less, which shouldn't happen but good to be safe),
    // there's nothing to rebalance! So, we only do the next steps if there's money.
    // if (portfolioBalance > 0) {

    // GOAL: We want our stocks to be, say, 60% of our total money,
    // and bonds to be 40%. This is our "target allocation".
    // `params.trStockAllocationRatio` holds our desired stock percentage (e.g., 0.60 for 60%).

    // Step 1: Calculate the IDEAL dollar amount we *should* have in stocks.
    // We take our current total portfolio value (`portfolioBalance`)
    // and multiply it by our target stock percentage.
    // Example: If portfolio is $100,000 and stock target is 60% (0.60),
    // then targetStock = $100,000 * 0.60 = $60,000.
    // This is the dream amount for stocks.
    const targetStock = portfolioBalance * params.trStockAllocationRatio;

    // Step 2: Calculate the IDEAL dollar amount we *should* have in bonds.
    // If stocks are 60%, then bonds must be 100% - 60% = 40%.
    // So, `(1 - params.trStockAllocationRatio)` gives us the bond percentage (e.g., 1 - 0.60 = 0.40).
    // Example: targetBond = $100,000 * 0.40 = $40,000.
    // This is the dream amount for bonds.
    const targetBond = portfolioBalance * (1 - params.trStockAllocationRatio);

    // Step 3: Figure out how much stock we need to BUY or SELL to hit our target.
    // We compare our `targetStock` (what we want) with our `stockBalance` (what we currently have
    // after this year's market movements).
    // - If targetStock is $60,000 and we only have $55,000 in stockBalance (maybe stocks didn't grow as much),
    //   then $60,000 - $55,000 = +$5,000. We need to BUY $5,000 worth of stock.
    // - If targetStock is $60,000 but we have $65,000 in stockBalance (maybe stocks grew a lot!),
    //   then $60,000 - $65,000 = -$5,000. We need to SELL $5,000 worth of stock.
    // We record this buy/sell amount in our logbook.
    logEntry.rebalanceStockAmount = targetStock - stockBalance;

    // Step 4: Do the same for bonds.
    // Figure out how much bond we need to BUY or SELL.
    // - If targetBond is $40,000 and we have $45,000 in bondBalance,
    //   then $40,000 - $45,000 = -$5,000. We need to SELL $5,000 of bonds.
    //   (This makes sense: if we're buying stock, we often sell bonds to get the cash, and vice-versa).
    // We record this in our logbook.
    logEntry.rebalanceBondAmount = targetBond - bondBalance;

    // Step 5: THE ACTUAL REBALANCE (in the simulation's world)
    // Now that we know our targets, we simply update our current balances to match these targets.
    // It's like the simulation magically makes the trades happen.
    // Our `stockBalance` is now officially set to the `targetStock` amount.
    stockBalance = targetStock;

    // And our `bondBalance` is now officially set to the `targetBond` amount.
    bondBalance = targetBond;

    // And that's it! Our portfolio is now perfectly back to our desired 60/40 (or whatever) split.
    // We've "taken profits" from the asset that did well and "bought the dip" in the asset that did less well,
    // bringing everything back in line.
    }
    // If the `if (portfolioBalance > 0)` was false, all these steps are skipped.


-----------------------------------------------
COLUMN HEADERS FOR 3-BUCKET STRATEGY EXPLAINED
-----------------------------------------------


| Column Header          | What it Represents                                                                                   |
| ---------------------- | ---------------------------------------------------------------------------------------------------- |
| **Year**               | Which year of the simulation you’re looking at (1, 2, 3, …).                                         |
| **Start Port.**        | Total portfolio value at the **beginning** of that year.                                             |
| **Withdrawal**         | Amount you pulled out for living expenses during that year.                                          |
| **B1 Start**           | Cash bucket (Bucket 1) balance at the start of the year.                                             |
| **B1 W/D**             | Portion of your withdrawal taken out of Bucket 1.                                                    |
| **B1 Ret%**            | Cash bucket’s percentage return that year (e.g., interest earned).                                   |
| **B1 Growth**          | Dollar amount the cash bucket gained or lost from returns.                                           |
| **B1 After Growth**    | Cash bucket balance **after** adding/subtracting growth.                                             |
| **B1 Target \$**       | How much cash you aim to keep in Bucket 1 (based on your “years of expenses” setting).               |
| **B1 Refill Amt**      | Dollars moved into Bucket 1 to bring it back up to its target.                                       |
| **B1 Refill Src**      | Where those refill dollars came from (e.g., from reallocating the overall portfolio or from B2).     |
| **B1 End**             | Cash bucket balance at the **end** of the year (after growth and any refilling).                     |
| **B2 Start**           | Bonds bucket (Bucket 2) balance at the start of the year.                                            |
| **B2 W/D**             | Portion of your withdrawal taken out of Bucket 2.                                                    |
| **B2 Ret%**            | Bond bucket’s percentage return that year.                                                           |
| **B2 Growth**          | Dollar amount the bond bucket gained or lost from returns.                                           |
| **B2 After Growth**    | Bond bucket balance after growth.                                                                    |
| **B2 Target \$**       | How much you aim to keep in Bucket 2 (based on its “years of expenses” setting).                     |
| **B2 To B1**           | Dollars transferred from Bucket 2 into Bucket 1 to refill it during down markets.                    |
| **B2 Rebal.**          | Dollars moved between B2 and B3 when rebalancing in up markets or after refills.                     |
| **B2 End**             | Bond bucket balance at the end of the year.                                                          |
| **B3 Start**           | Stock bucket (Bucket 3) balance at the start of the year.                                            |
| **B3 W/D**             | Portion of your withdrawal taken out of Bucket 3.                                                    |
| **B3 Ret% (Decision)** | Stock bucket’s return **used to decide** whether markets were “up” or “down” for rebalancing.        |
| **B3 Growth**          | Dollar amount the stock bucket gained or lost from returns.                                          |
| **B3 After Growth**    | Stock bucket balance after growth.                                                                   |
| **B3 To B1**           | Dollars moved from Bucket 3 into Bucket 1 (only if both B1 & B2 are empty).                          |
| **B3 Rebal.**          | Dollars moved from B2 into B3 (in down markets) or from B3 into B2 (in up markets) during rebalance. |
| **B3 End**             | Stock bucket balance at the end of the year.                                                         |
| **Realloc Strat.**     | Which refill/rebalancing rule was applied that year (e.g., “Market Up/Flat” or “Market Down”).       |
| **End Port.**          | Total portfolio value at the **end** of the year (sum of B1+B2+B3).                                  |
| **Next Yr W/D**        | How much your annual withdrawal will be next year (inflation-adjusted).                              |


----------------------------------------------------
COLUMN HEADERS FOR TOTAL RETURN STRATEGY EXPLAINED
----------------------------------------------------


| Column Header          | What it Represents                                                                             |
| ---------------------- | ---------------------------------------------------------------------------------------------- |
| **Year**               | Which year of the simulation you’re looking at (1, 2, 3, …).                                   |
| **Start Port.**        | Total portfolio value at the **beginning** of that year.                                       |
| **Withdrawal**         | Amount you pulled out for living expenses during that year.                                    |
| **Port. After W/D**    | Portfolio value **after** you’ve taken that withdrawal (before any growth).                    |
| **Stock Start**        | Portion invested in stocks at the start of the year.                                           |
| **Stock W/D**          | Portion of your withdrawal taken out of the stock bucket.                                      |
| **Stock Ret%**         | Stock bucket’s percentage return that year.                                                    |
| **Stock Growth**       | Dollar amount the stock bucket gained or lost from returns.                                    |
| **Stock After Growth** | Stock bucket balance **after** adding/subtracting growth.                                      |
| **Stock Rebal.**       | Dollars moved into or out of stocks to reset back to your target allocation (e.g. 60% stocks). |
| **Stock End**          | Stock bucket balance at the **end** of the year (after growth and rebalancing).                |
| **Bond Start**         | Portion invested in bonds at the start of the year.                                            |
| **Bond W/D**           | Portion of your withdrawal taken out of the bond bucket.                                       |
| **Bond Ret%**          | Bond bucket’s percentage return that year.                                                     |
| **Bond Growth**        | Dollar amount the bond bucket gained or lost from returns.                                     |
| **Bond After Growth**  | Bond bucket balance **after** adding/subtracting growth.                                       |
| **Bond Rebal.**        | Dollars moved into or out of bonds during the annual rebalance.                                |
| **Bond End**           | Bond bucket balance at the **end** of the year.                                                |
| **End Port.**          | Total portfolio value at the **end** of the year (sum of stocks + bonds).                      |
| **Next Yr W/D**        | How much your annual withdrawal will be next year (inflation-adjusted).                        |
