Simple note used to store important resources for the use and maintenance of my [Kyria v3](https://splitkb.com/products/kyria-rev3?srsltid=AfmBOoqMxrL8O1bULN0UNz34XBI0CywqDGOFLghDUuBZhv_GDEFcyeck) 
## Parts and Guides

The Kyria v3 followed the simple Aurora [build guide](https://docs.splitkb.com/product-guides/aurora-series/build-guide). Notably the [nice!view](https://nicekeyboards.com/docs/nice-view/getting-started) I got uses SPI as such I just had to solder the chip select (cs) pin to some open pin on the nice!nano. The pin I chose was the p1.01 as recommended by Araxia on Discord [here](https://discordapp.com/channels/574598631399751680/1139169271168114811), Araxia seems to have even made a small blog [post](https://araxia.net/keyboards/nice-view-bodge/) covering this.

I am building ZMK using the github actions for now, my actions page can be found at [zmk-config](https://github.com/MatthewWinnan/zmk-config/actions)

Good Kyria [Schematics](https://docs.splitkb.com/product-guides/kyria/schematics) of course you can always check out the main [github](https://github.com/splitkb/kyria)

I am using [ZMK](https://zmk.dev/docs) so please consult their guide for general configurations.
## Keymaps

Some ideas I have come across:

- https://nicekeyboards.com/docs/nice-view/getting-started
- https://www.reddit.com/r/ErgoMechKeyboards/comments/v28xs0/kyria_keylayout_feedback_needed/
