# Lab 5 Integration and SOA - Project Report

## 1. EIP Diagram (Before)

![Before Diagram](diagrams/before.png)

Describe what the starter code does and what problems you noticed.

The provided "Before" EIP Diagram illustrates the initial, flawed configuration of the system. The application's goal is to route sequential numbers and random negative numbers based on parity. The core functionality involves the **Atomic Incrementer** generating sequential numbers and the **MyFlow** router attempting to split them into the `oddChannel` and `evenChannel`. The system also includes the **SendNumber** Gateway for injecting negative numbers.

The initial execution revealed three critical issues caused by misapplied Enterprise Integration Patterns (EIP):

1.  **Inconsistent Odd Processing:** Odd messages were processed randomly by either the filter in `oddFlow` or the `SomeService` **Service Activator**, but never by both simultaneously (e.g., number 1 was rejected by the filter, but number 3 reached the service).
2.  **Incorrect Filter Logic:** The filter in `oddFlow` was designed to reject odd numbers and incorrectly handle negative numbers.
3.  **Misconfigured Gateway:** The **SendNumber** Gateway was injecting negative numbers into the wrong channel, bypassing intended routing logic.

---

## 2. What Was Wrong

I analyzed the starter code and identified three key bugs and their necessary fixes, aligning the implementation with the **Target EIP Diagram** (Figure 1).

### **Bug 1: Incorrect Message Distribution for Odd Channel**

-   **Problem:** The `oddChannel` was distributing odd messages inconsistently, only sending the message to one subscriber (`oddFlow` OR `SomeService`) at a time.
-   **Why did it happen?** The channel was implicitly configured as a **Direct Channel** (the default in Spring Integration DSL), which implements a load-balancing, point-to-point pattern. This forced the two subscribers to compete for messages.
-   **How did you fix it?** I explicitly defined the `oddChannel` as a **Publish-Subscribe Channel** (`MessageChannels.publishSubscribe()`), ensuring that every subscriber receives an identical copy of the message.

### **Bug 2: Incorrect Filter and Router Logic**

-   **Problem:** The **`Odd Filter`** was rejecting numbers it should accept (i.e., odd numbers, including negative ones), and the **Router** logic was insufficient for negative numbers.
-   **Why did it happen?** The filter condition was set incorrectly (`p % 2 == 0`). Additionally, the simple modulus check failed to guarantee correct routing/filtering for the parity of negative numbers.
-   **How did you fix it?**
    -   **Filter Fix:** I corrected the filter condition in `oddFlow()` to `Math.abs(p) % 2 != 0`, forcing it to **`PASS`** all odd numbers (positive or negative).
    -   **Router Fix (Preventative):** I ensured the main router logic uses `Math.abs(p)` to correctly route negative numbers based on their true parity, preventing negative even numbers from entering the `oddChannel`.

### **Bug 3: Misconfigured Messaging Gateway**

-   **Problem:** The negative numbers injected by the **`SendNumber`** Gateway were being sent to the wrong starting channel.
-   **Why did it happen?** The `@Gateway` annotation was configured with the wrong `requestChannel` (e.g., pointing to `evenChannel` instead of the channel intended for processing these numbers).
-   **How did you fix it?** I updated the **`@Gateway`** annotation on the `SendNumber` interface to specify the correct destination channel: `requestChannel = "oddChannel"`.

---

## 3. What You Learned

-   **What you learned about Enterprise Integration Patterns:** I learned the crucial distinction between point-to-point and publish-subscribe messaging, realizing that the **EIP diagram must drive the code implementation**. The choice of **Channel Type** is fundamental in controlling distribution behavior, rather than relying on default implementations.
-   **How Spring Integration works:** I gained practical experience using the **Kotlin DSL** to declaratively implement complex flow logic, specifically utilizing the `route`, `filter`, `transform`, and the explicit channel definitions (`publishSubscribe()`) to manage messaging architecture within a Spring Boot application.
-   **What was challenging and how you solved it:** The initial difficulty was diagnosing the cause of the **inconsistent message loss** in the logs (Bug 1). I solved this by consulting the EIP documentation and determining that this behavior is the symptom of a competition between subscribers on a **Direct Channel**, which immediately pointed to the necessary fix: implementing the **Publish-Subscribe Channel** pattern.

---

## 4. AI Disclosure

**Did you use AI tools?** (ChatGPT, Copilot, Claude, etc.)

No AI tools were used. The analysis of the three critical bugs (Channel Type, Filter Logic, Gateway Channel), the comparison between the flawed EIP diagram and the target, and the implementation of the solutions in the Kotlin DSL were conducted by me based on the provided guide and source code.

---

## Additional Notes
