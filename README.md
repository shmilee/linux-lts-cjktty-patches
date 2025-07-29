# linux-LTS-cjktty-patches

Patches from [zhmars/cjktty-patches](https://github.com/zhmars/cjktty-patches)
and [Gentoo-zh/linux-cjktty](https://github.com/Gentoo-zh/linux-cjktty) with minor changes.

- Starting with linux LTS 6.6, the kernel config option `CONFIG_FONT_16x16_CJK` has been renamed to `CONFIG_FONT_CJK_16x16`.
- To have a larger font on high resolution screens, you probably want to apply 32x32 font data patch.
- The patch built-in fonts expect to work with 8x16 or 16x32 fonts. When changing to other font sizes, characters may not display correctly.

## Changes

### 2025.07.xx / 6.12

- Resync for linux-6.12.y

### 2023.10.30 / 6.6 (upstream)

- Resync for linux-6.6.y
- Update font data to Unifont 15.1.04
- Update double width tables to Unicode 15.1.0

## Credits

- [zhmars/cjktty-patches](https://github.com/zhmars/cjktty-patches) for v3.x v4.x v5.x and v6.0 to v6.7 cjktty patches
- [youbest](http://blog.chinaunix.net/uid/436750.html) for [original univt patches](https://github.com/zhmars/univt-patches/tree/master/v2.6)
    - [补丁原文整理](https://zhuanlan.zhihu.com/p/486655122)
- [Gentoo-zh/linux-cjktty](https://github.com/Gentoo-zh/linux-cjktty) for [original cjktty patches](https://github.com/torvalds/linux/compare/master...Gentoo-zh:linux-cjktty:5.6-utf8)
- [AOSC-Dev/aosc-os-abbs](https://github.com/AOSC-Dev/aosc-os-abbs) for some univt's modifications
- [Unifont](https://savannah.gnu.org/projects/unifont) for font data
- [Terminus Font](http://terminus-font.sourceforge.net) for font data

