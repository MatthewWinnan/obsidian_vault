What follows is a general overview of potential future projects I want to do, or things I still need to work on for various topics.
## Language Learning Projects

- [ ] Learn RUST from the [RUST Book](https://doc.rust-lang.org/book/)
	- [ ] Currently at [7.4](https://doc.rust-lang.org/book/ch07-04-bringing-paths-into-scope-with-the-use-keyword.html)
- [ ] Learn Zig [Zig-Learn](https://ziglang.org/learn/)
- [ ] I ideally want to become more familiar with Nix, like I can comfortably perform known tasks, I can't however solve novel problems. 
	- [ ] Starting might be [Nix-Pills](https://nixos.org/guides/nix-pills/)

## Mech Keeb And ZMK

- [ ] I still need to get keymap viewer working for the  TIE-TEM.
- [ ] Finish flashing my newest keymap and test it, for both the kyria and TIE-TEM.
- [ ] Finish my TIE-TEM write up and add images and files on how it was made printed etc. Also need to give the appropriate credits.
- [ ] Currently my TIE-TEM was incorrectly assembled, I would love to just have a per config overlay of sorts to remap the row definition only for left side, so the main TIE-TEM module I am pulling in can be left with the correct mapping.
- [ ] Learn more about Zephyr, for example I have an I2C rotary encoder which would be nice to integrate so I do not need extra pins (would allow for things like TIE-TEM to have encoders)
	- [ ] First learn how the rotary encoder currently functions.
	- [ ] Learn how device tree files are structured so I can make my own.
	- [ ] How I should expose the module so it can be imported and configured.
- [ ] Finish up my RGB project, or check how far the current implementation is, it was some series of PR's which implemented it for the glove and generalized it for ZMK.
- [ ] Plan my next board?

## 3D Printer

- [ ] Need to send it in for service
	- [ ] This should address the noises when going in the positive Y direction.
	- [ ] I should also address the X-axis movements being rather resistent.
- [ ] Print potplant Junimo for GF [Junimo 1](https://www.printables.com/model/208635-stardew-valley-junimo-vase-for-plant-pot/files), [Junimo 2](https://www.printables.com/model/233488-stardew-valley-junimo-succulent-holder)

## NixOS

- [ ] I want to get movcli to work on my system.
- [ ] I saw a new project [RMPC](https://github.com/mierak/rmpc)
- [ ] I still need to finally port my main 25.05 config to the work laptop.
- [ ] Get a nice workflow for RUST based development, I suppose just using cargo is enough and having that on system? I won't be installing just running. I suppose the question is does it just need to get the compiler's PATH or is there a runtime of sorts?
- [ ] I want to get the big RTX3070 to work on NixOS so I can fully integrate into the Linux community.
- [ ] Add local nix-caching using [attic](https://github.com/zhaofengli/attic).
- [ ] Maybe I should add Hashtech Vault?
	- [ ] To this extent would Vault Guard be a good idea to keep my passwords for local use?
- [ ] Revive my own [SearchXNG](https://github.com/searxng/searxng).