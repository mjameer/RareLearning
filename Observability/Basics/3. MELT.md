Observability platforms collect four types of telemetry data, often remembered by the acronym **MELT**:

* **M - Metrics:** An aggregated value representing events over a period of time. Metrics are good for comparing system performance.
    * *Vending Machine Example:* 100 bags of chips were purchased every minute. This can be compared to last week's rate of 200 bags per minute to spot a decline.
* **E - Events:** A record of a specific action that happened at a given time. Events validate that an expected action occurred.
    * *Vending Machine Example:* A customer purchased a bag of chips at 3:20 p.m.
* **L - Logs:** A very detailed representation of an event, containing much more contextual information.
    * *Vending Machine Example:* A bag of chips ($2) was purchased at 3:20 p.m. from machine ID X in Sydney using a Mastercard.
* **T - Traces:** Shows the interaction and path of a request as it travels through multiple microservices in a distributed system. Traces are crucial for identifying where a failure occurred in a complex interaction.
    * *Vending Machine Example:* A trace would show the request flow from the vending machine to the bank, then to Mastercard, and the responses back, to complete a credit card purchase.
 
    * https://notebooklm.google.com/notebook/795ca786-fc40-4e74-b118-cc86baba69f8
