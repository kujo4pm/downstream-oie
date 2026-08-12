# Intro
This is a Open Integration Engine configuration that's intended to complement SENAITE's FHIR API, with the Implementation Guide: https://fhir.senaite.org/. Specifically the [Instrument Integration workflow](https://fhir.senaite.org/instrument-integration.html). In short -- it is intended to demonstrate how instruments using an HL7 or even ASTM workflow can use OIE is a middleware layer to translate to FHIR.

## Architectural Overview

We will create the following channels:
- `fetch-token`: This will log into SENAITE's FHIR API and retrieve a token and save it to Global Maps
- `fetch-instrument-worklist`: This fetches all of SENAITE's Instrument Service Requests. That is all analyte Service Requests intended for Instruments. This will then save these to the middleware Postgres database.
- `dispatch-instrument-orders`: This fetches the ServiceRequest from the database and pushes it as a HL7 order to the instrument.
- `receive-instrument-results`: This receives the results from the instrument and saves the variables into the OIE database.
- `post-observations-to-fhir`: This polls the middleware database for new results. This translates them into a SENAITE Observation and POSTs the results.


## Example Run
Steps:

1. Create a new Sample in Senaite instance - for example to test Electrolytes.
2. For Chloride levels assign to an instrument blood analyzer.
3. Set Sample Received. This will create the Instrument Service Request fetched by OIE and saved to the database.
4. 