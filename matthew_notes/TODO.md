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
- [ ] I do not like my Kyria_View naming convention, this only implies one board, rather use my name to imply it is my fork.
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

- [x] I want to get movcli to work on my system. ✅ 2025-05-25
	- [x] Get some plugins going ✅ 2025-05-25
		- [x] Youtube ✅ 2025-05-25
		- [x] Anime ✅ 2025-05-25
		- [x] Shows ✅ 2025-05-25
			- [x] Using [Lobster](https://github.com/justchokingaround/lobster?tab=readme-ov-file#nixos-flake) ✅ 2025-05-25
		- [x] Movies ✅ 2025-05-25
			- [x] Using [Lobster](https://github.com/justchokingaround/lobster?tab=readme-ov-file#nixos-flake) ✅ 2025-05-25
- [x] I saw a new project [RMPC](https://github.com/mierak/rmpc) ✅ 2025-06-01
	- [ ] I need to customize it and figure out how to organize music.
	- [ ] Get a nice music downloader, currently downloading from youtube with yt-dlp
- [ ] I still need to finally port my main 25.05 config to the work laptop.
- [ ] Get a nice workflow for RUST based development, I suppose just using cargo is enough and having that on system? I won't be installing just running. I suppose the question is does it just need to get the compiler's PATH or is there a runtime of sorts?
- [ ] I want to get the big RTX3070 to work on NixOS so I can fully integrate into the Linux community.
- [ ] Look at alternative LSP for Nix?
	- [ ] There was a good talk about analysis for the language, however in terms of LSP you'll always be limited it seems.
- [ ] I have noticed waypaper using hyprpaper backend does not always launch the restore function, try to fix.
- [x] Look at instead using librewolf, instead of Schizofox as default browser. ✅ 2025-05-25
	- [x] Maybe even rather qutebrowser ✅ 2025-05-25
	- [x] Or maybe even nyxt ✅ 2025-05-31
	- [ ] I want to see what else I can customize.
		- [ ] Transparent tabs.
		- [ ] Filemanager color and picker.
	- [x] I still need to make it my default browser ✅ 2025-06-07
- [ ] Make orcaslicer more reproducible? 
	- [ ] Build from source
	- [ ] Symlink configs and keep configs in git repo?
- [ ] Add some applications as systemd services
	- [ ] Waybar.
	- [ ] Clipboard.
	- [ ] A lot of processes I have hyprland start and detach from could probably made services or one shot systemd tasks.
- [ ] Hyprland fixes so most things works well enough with xwayland and so forth.
- [ ] I want to change some waybar things, make it sharper cleaner maybe transparent. 
	- [ ] Look at some examples.
	- [ ] Same for hyprland windows?
- [ ] Learn specializations.
- [ ] Learn overlays and overrides more, have an idea but in context of NixOS not sure.
	- [ ] Also is it opt in or can any attribute set take an overlay to change the contents?
	- [ ] Maybe pills helps.e

## Home Lab

- [ ] Revive my own [SearchXNG](https://github.com/searxng/searxng).
- [ ] Add local nix-caching using [attic](https://github.com/zhaofengli/attic).
	- [x] Want to make the delta my server. ✅ 2025-06-08
		- [ ] Ended up using proxmox on my H3+
	- [x] Need to change it to be headless. ✅ 2025-06-08
	- [x] Add headless options into my config? ✅ 2025-06-08
	- [x] Need to also split things I import between headless and so forth. ✅ 2025-06-08
	- [ ] Get a atticd service going, solved this for work read and base off of write up there
- [ ] Add a wallpaper repository on github and import it as a flakes input.
- [ ] Start a media centre
	- [ ] [Plex Server](https://www.plex.tv/)
		- [ ] Integrate mov-cli with it mayhaps -> [mov-cli-jellyplex](https://github.com/mov-cli/mov-cli-jellyplex)
		- [ ] [Tutorial Video [1]](https://www.howtogeek.com/did-you-know-you-could-stream-plex-or-jellyfin-in-your-terminal/)
	- [ ] [Jellyfin](https://jellyfin.org/)
		- [ ] https://www.youtube.com/watch?v=FpFFnTH6-Ww
- [ ] Maybe I should add Hashtech Vault?
	- [ ] To this extent would Vault Guard be a good idea to keep my passwords for local use?
- [ ] Revive my Home Assistant and get actual information.
	- [ ] First figure out how to send data via MQTT
	- [ ] Decide on sensors to use, MCU will just be one of my pico W's laying about
- [ ] GF uses a library service to host public books, maybe get that up and running too.
	- [ ] Find out the name.
	- [ ] Integrate mov-cli with it if possible?
- [ ] Get a local DNS server going for my own personal systems
- [ ] Get a local dashboard going for basic navigation
	- [ ] Maybe also SNMP up down monitoring?
- [ ] Get a local nightly mirror of Nixpkgs going? 

## Treasure Trove

The following tools look good and I want to read more on how to integrate them to my workflow.

- [atuin](https://atuin.sh/)
	- Nix Options -> https://mynixos.com/search?q=atuin   


