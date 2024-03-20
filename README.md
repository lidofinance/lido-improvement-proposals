# [LIPs](https://research.lido.fi/) 

> Lido Improvement Proposals (LIPs) describe standards for the Lido platform, including core protocol specifications, client APIs, and contract standards.
 
## Contributing

 1. Review [LIP-0](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-0.md).
 2. Fork the repository by clicking "Fork" in the top right.
 3. Add your LIP to your fork of the repository. There is a [template LIP here](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/lip-X.md).
 4. Submit a Pull Request to lido's [LIPs repository](https://github.com/lidofinance/lido-improvement-proposals).

Your first PR should be a first draft of the final LIP. It must meet the formatting criteria enforced by the build (largely, correct metadata in the header). An editor will manually review the first PR for a new LIP and assign it a number before merging it. Make sure you include a `discussions-to` header with the URL to a new thread on [research.lido.fi](https://research.lido.fi/) where people can discuss the LIP as a whole.

If your LIP requires images, the image files should be included in a subdirectory of the `assets` folder for that LIP as follow: `assets/Lip-X` (for Lip **X**). When linking to an image in the LIP, use relative links such as `../assets/Lip-X/image.png`.

When you believe your LIP is mature and ready to progress past the WIP phase, you should ask to have your issue added to the next governance call where it can be discussed for inclusion in a future platform upgrade. If the community agrees to include it, the LIP editors will update the state of your LIP to 'Approved'.

## LIP Statuses

* **WIP** - a LIP that is still being developed.
* **Proposed** - a LIP that is ready to be reviewed in a governance call.
* **Approved** - a LIP that has been accepted for implementation by the Lido community.
* **Implemented** - a LIP that has been released to mainnet.
* **Rejected** - a LIP that has been rejected.
* **Withdrawn** - a LIP that has been withdrawn by the author(s).
* **Deferred** - a LIP that is pending another LIP/some other change that should be bundled with it together.
* **Moribund** - a LIP that was implemented but is now obsolete and requires no explicit replacement.

>**Note**   
> Diagram of LIP Progressions 

```mermaid
stateDiagram-v2
  direction LR
  [*] --> Draft
  Draft --> Review
  Review --> Living
  Review --> Implementation
  Implementation --> Final
  Final --> [*]
  Final --> Moribund
  
  Draft --> Withdrawn
  Review --> Withdrawn
  Implementation --> Withdrawn
  Implementation --> Deferred
  Withdrawn --> [*]
```


## Validation

LIPs must pass some validation tests. The LIP repository ensures this by running tests using [html-proofer](https://rubygems.org/gems/html-proofer) and [lip_validator](https://rubygems.org/gems/lip_validator).

It is possible to run the LIP validator locally:
```
gem install lip_validator
lip_validator <INPUT_FILES>
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
