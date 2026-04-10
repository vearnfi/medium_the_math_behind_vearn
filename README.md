
# The Math Behind Vearn Finance: Optimizing VTHO → VET Swaps Over the Long Run

> **Note.** The Vearn protocol was developed under the pre-Hayabusa VeChain tokenomics model. Since Hayabusa, VTHO generation and fee mechanics have changed, so this analysis is no longer applicable to the current system.

---

## 1. Introduction

If you hold VET, you already know one of the most interesting properties of VeChain's tokenomics: simply holding VET generates VTHO over time.

VTHO is required to pay for transactions on the network, but if you accumulate more than you actually need, the surplus becomes idle capital. That's where **Vearn Finance** comes in.

Vearn Finance is designed to put that surplus to work by converting excess VTHO into more VET. To make that loop efficient, however, one question becomes central: *when is the right time to swap?* This article explains the mathematical model Vearn uses to answer that question.

---

## 2. The Core Idea

The objective of Vearn is to leverage VeChain's tokenomics to grow a user's VET balance over the long run. The idea is to swap the VTHO generated from holding VET for additional VET, creating a positive feedback loop: more VET generates more VTHO, which can then be swapped for even more VET.

The question is when those swaps should happen in order to maximize the final VET balance.

If swaps happen as soon as VTHO is generated, the user pays too much in fees. At the other extreme, swapping only rarely reduces fees, but the VET balance also grows more slowly. The optimal strategy lies somewhere in between.

This leads to two natural approaches: swap VTHO for VET at fixed time intervals, or swap whenever the wallet accumulates a certain amount of VTHO. In Vearn Finance, we focus on the second approach: a swap is executed whenever the VTHO balance reaches a value we denote by **swapThreshold**.

### How should swapThreshold be chosen?

In principle, the optimal threshold depends on the user's current VET balance, the VTHO/VET exchange rate, and the current fee levels - all of which change over time. Since we cannot forecast how these quantities will evolve, we instead take a snapshot approach: we treat current market conditions as fixed, and ask which threshold would perform best *if those conditions were to persist*. This is a deliberate modeling choice, not an assumption that markets are stable.

Concretely, we analyze the long-run evolution of the user's VET balance under a fixed threshold and fixed market conditions, and repeat this analysis across a range of candidate thresholds. The threshold that yields the highest final VET balance is selected as the optimal one for the current snapshot.

### How is this applied in practice?

Rather than fixing the threshold forever, the protocol reruns this analysis at regular intervals using the user's current VET balance and the latest market conditions. At each iteration, if the user's available VTHO balance is greater than or equal to the newly computed threshold, a swap is executed. Otherwise, the system waits and recalculates at the next cycle. This rolling recalibration allows the strategy to adapt continuously to changing conditions - without ever needing to forecast them.

The rest of this article explains the mathematical model used to estimate that threshold.

---

## VTHO Generation Rate

Holding VET generates VTHO at a predictable rate. Let **vthoGen** denote the amount of VTHO generated per unit of VET per second:

**vthoGen = 5 · 10⁻⁹ VTHO / (VET · sec)**

Under this model, a user holding **VET₀** generates VTHO at a constant rate of **vthoGen · VET₀** per second. As long as the VET balance stays fixed, the user's VTHO balance therefore grows linearly over time.

---

## 3. Balance Growth and Swap Timing

We now move from the intuition of the strategy to a mathematical model of how the user's balance evolves over time.

As established in Section 2, we study this evolution under fixed market conditions, assuming that a swap is executed whenever the VTHO balance reaches a fixed **swapThreshold**. To describe the outcome of each swap, we need two additional quantities: let **vthoFee** denote the VTHO spent on fees per swap, and let **vetPerVtho** denote the VTHO → VET exchange rate. This is the first step toward finding the optimal threshold for a given snapshot of market conditions.

Let time start at **t₀**, and assume the initial VET balance is:

**VET₀ > 0**

Let **tᵢ** denote the time of the i-th swap, and let **VETᵢ** denote the user's VET balance after the i-th swap.

### VET Balance After Each Swap

When a swap occurs, the amount of VET gained is the net VTHO after fees, converted at the current exchange rate:

**ΔVET = (swapThreshold − vthoFee) · vetPerVtho**  **(1)**

Since **swapThreshold** is fixed and, by equation **(1)**, each swap yields the same **ΔVET** as long as **vetPerVtho** and **vthoFee** remain constant, the user's VET balance increases by the same amount at every swap.

After the first swap:

**VET₁ = VET₀ + ΔVET**

After the second swap:

**VET₂ = VET₁ + ΔVET = VET₀ + 2 · ΔVET**

More generally, after the i-th swap:

**VETᵢ = VET₀ + i · ΔVET**  **(2)**

Equation **(2)** shows that, under fixed conditions, the VET balance grows linearly with the number of swaps.

### Time Between Swaps

At time **t₀**, the user holds **VET₀** and generates VTHO at a rate of **vthoGen · VET₀**. The time needed to accumulate enough VTHO to reach **swapThreshold** is therefore:

**t₁ − t₀ = swapThreshold / (vthoGen · VET₀)**

More generally, after the i-th swap, the user holds **VETᵢ**, so VTHO is generated at a rate of **vthoGen · VETᵢ**. The time until the next swap is then:

**tᵢ₊₁ − tᵢ = swapThreshold / (vthoGen · VETᵢ)** **(3)**

As **VETᵢ** grows, the time between swaps shrinks. This is the positive feedback loop described earlier becoming visible in the math: each swap increases the VET balance, which increases the VTHO generation rate, which makes the next swap happen sooner.

---

## Estimating the VET Balance Over a Long Time Horizon

To find the optimal **swapThreshold** for a given user under a given snapshot of market conditions, we need to evaluate the final VET balance across a range of candidate thresholds and select the one that performs best. Doing this naively, by simulating every swap one by one over a long time horizon, is computationally expensive even for a single candidate. Repeating that process across hundreds of threshold values, for thousands of users, at every cycle, quickly becomes impractical.

What we need instead is a closed-form expression: a formula that takes a snapshot of market conditions, a time interval **T** and a candidate **swapThreshold** as inputs and returns the estimated final VET balance directly, without simulating each swap individually. Deriving that expression is the goal of this section.

Equation **(2)** tells us that, under fixed conditions, each swap adds the same amount **ΔVET** to the user's balance. This means the long-run problem reduces to a counting problem: if we can estimate how many swaps (**n**) occur within a time interval **T** for a given **swapThreshold**, then we can also estimate the final VET balance at the end of that interval.

Let **T** be the total time interval we want to analyze:

**T = tₙ − t₀**

Since **T** is the sum of all time gaps between swaps, we have:

**T = Σᵢ₌₀ⁿ⁻¹ (tᵢ₊₁ − tᵢ)**

Substituting the expression for each interval (equation **(3)**):

**T = Σᵢ₌₀ⁿ⁻¹ swapThreshold / (vthoGen · VETᵢ)**

Factoring out the constants:

**T = (swapThreshold / vthoGen) · Σᵢ₌₀ⁿ⁻¹ (1 / VETᵢ)**

Using equation **(2)**, we can rewrite the sum as:

![Rewritten formula](imgs/rewritten-formula.png)

This is a shifted harmonic series. When **n** is large, which is precisely the regime this strategy targets, we can approximate the sum with a logarithmic expression:

![Log approximation formula](imgs/log-approximation-formula.png)

Here, **γ** is related to the Euler-Mascheroni constant and tends to 0 in our approximation.

This approximation is tightest precisely where the strategy is most useful: users with meaningful VET balances over long time horizons. Users with very small VET balances will generate few swaps over any given interval, so the model naturally reflects that there is little to optimize in that case.

Rearranging:

![Rearranged formula](imgs/rearranged-formula.png)

Solving for **n**:

![Solved n formula](imgs/solved-n-formula-v2.png)

With this approximation in hand, the problem becomes computationally tractable. For any candidate **swapThreshold**, we can estimate how many swaps **n** occur over the interval **T**, and from that estimate the corresponding final VET balance. This gives us a practical way to evaluate many possible threshold values efficiently and identify the one that produces the best result for the current snapshot.

## Autopilot: From Theory to Product

Vearn Finance packages this approach into a simple, passive experience. Users choose a reserve balance — the amount of VTHO they want to keep in their account at all times — and grant the Vearn smart contract allowance to spend the rest. From there, the following procedure runs automatically at each cycle:

1. **Fetch inputs.** The optimization algorithm retrieves the user's current VET balance, the network gas price, and the VTHO/VET exchange rate.
2. **Compute the optimal threshold.** Using the closed-form approximation derived above, the algorithm evaluates a range of candidate thresholds and selects the one that maximizes long-run VET growth under current conditions.
3. **Apply the threshold strategy.** If the user's available VTHO balance (total minus reserve) is greater than or equal to the optimal **swapThreshold**, a swap is executed. Otherwise, the system waits.
4. **Repeat.** The procedure restarts at the next cycle with updated inputs.

The result is a disciplined and automatic strategy for increasing VET holdings over time, without requiring the user to time the market.

---

## Conclusion

The central idea behind Vearn is simple: if holding VET generates VTHO, then the right swap policy can turn that passive flow into long-term VET growth. The challenge is not whether to swap, but when.

Under a snapshot of current market conditions, the problem can be modeled mathematically. By combining the per-swap gain in equation **(1)**, the balance evolution in equation **(2)**, and the approximation for the number of swaps in equation **(3)**, we obtain a practical way to estimate the final VET balance for any candidate **swapThreshold**.

That is what makes the strategy usable in practice. Rather than relying on market forecasts or expensive simulation, the protocol can repeatedly evaluate current conditions, compute the threshold that best fits them, and apply the same rule again at the next cycle. The assumption of fixed market conditions is therefore local to each run of the analysis, not a claim about the whole future path of the market. In that sense, the system is not trying to predict the future - it is simply making the best decision it can with the information available at each step.

---

## Links

* App: [https://vearn.finance](https://vearn.finance)
* Open source: [https://github.com/vearnfi](https://github.com/vearnfi)
* Original announcement post: [https://medium.com/@vearnfi/grow-your-vet-balance-with-vearn-finance-a227d2b02de9](https://medium.com/@vearnfi/grow-your-vet-balance-with-vearn-finance-a227d2b02de9)
* X / Twitter: [https://x.com/fede_rodes](https://x.com/fede_rodes)

---

## Interested in Working Together?

If you are looking for a full-stack web3 developer with a background in applied math, feel free to reach out through the project links above. Vearn was built at the intersection of product engineering, smart contract integration, and mathematical modeling, and I would be happy to discuss similar opportunities.
