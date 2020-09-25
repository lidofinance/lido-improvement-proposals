# DIPs [![Discord](https://img.shields.io/discord/734804446353031319.svg?color=768AD4&label=discord&logo=https%3A%2F%2Fdiscordapp.com%2Fassets%2F8c9701b98ad4372b58f13fd9f65f966e.svg)](https://discordapp.com/channels/734804446353031319/) [![Telegram](https://img.shields.io/badge/chat-on%20Telegram-blue.svg)](https://t.me/yearnfinance) [![Twitter Follow](https://img.shields.io/twitter/follow/iearnfinance.svg?label=iearnfinance&style=social)](https://twitter.com/iearnfinance)

Deepol Improvement Proposals (DIPs) describe standards for the yEarn platform, including core protocol specifications, client APIs, and contract standards.
 
## Contributing

 1. Review [DIP-0](DIPS/Dip-0.md).
 2. Fork the repository by clicking "Fork" in the top right.
 3. Add your DIP to your fork of the repository. There is a [template DIP here](Dip-X.md).
 4. Submit a Pull Request to yEarn's [DIPs repository](https://github.com/depools/depools-improvement-proposals).

Your first PR should be a first draft of the final DIP. It must meet the formatting criteria enforced by the build (largely, correct metadata in the header). An editor will manually review the first PR for a new DIP and assign it a number before merging it. Make sure you include a `discussions-to` header with the URL to a new thread on [gov.yearn.finance](https://gov.yearn.finance/) where people can discuss the DIP as a whole.

If your DIP requires images, the image files should be included in a subdirectory of the `assets` folder for that DIP as follow: `assets/Dip-X` (for Dip **X**). When linking to an image in the DIP, use relative links such as `../assets/Dip-X/image.png`.

When you believe your DIP is mature and ready to progress past the WIP phase, you should ask to have your issue added to the next governance call where it can be discussed for inclusion in a future platform upgrade. If the community agrees to include it, the DIP editors will update the state of your DIP to 'Approved'.

## DIP Statuses

* **WIP** - a DIP that is still being developed.
* **Proposed** - a DIP that is ready to be reviewed in a governance call.
* **Approved** - a DIP that has been accepted for implementation by the yEarn community.
* **Implemented** - a DIP that has been released to mainnet.
* **Rejected** - a DIP that has been rejected.
* **Withdrawn** - a DIP that has been withdrawn by the author(s).
* **Deferred** - a DIP that is pending another DIP/some other change that should be bundled with it together.
* **Moribund** - a DIP that was implemented but is now obsolete and requires no explicit replacement.

## Validation

DIPs must pass some validation tests.  The DIP repository ensures this by running tests using [html-proofer](https://rubygems.org/gems/html-proofer) and [yip_validator](https://rubygems.org/gems/yip_validator).

It is possible to run the DIP validator locally:
```
gem install yip_validator
yip_validator <INPUT_FILES>
```

## Copyright

Copyright and related rights waived via [DIPS](https://github.com/iearn-finance/DIPS/).
