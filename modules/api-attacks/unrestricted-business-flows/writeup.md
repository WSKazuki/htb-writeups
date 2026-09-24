# API6 — Unrestricted Access to Sensitive Business Flows

**Answer:** 788 Sauchiehall St. (Glasgow, UK — customerID daa8c984-ba84-4265-8d88-12d6607e51c)

## Method

No new exploit needed. The billing addresses response from the BFLA exercise
(GET /api/v1/customers/billing-addresses with a zero-role JWT) already contained
the target customer's full PII. The same unauthorized call satisfies both API5
and API6 findings simultaneously.

## Lesson

API6 is the business-impact layer on top of API5. One missing [Authorize]
attribute = BFLA + business flow abuse + PII exposure in a single response.
