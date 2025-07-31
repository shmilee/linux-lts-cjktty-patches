# tools

* patch
* diffutils
* patchutils version 0.4.2
* wget, gvim, delta

# Split example patches

## v6.6 (LTS)

```
wget -c https://github.com/zhmars/cjktty-patches/raw/b43d618da6d6536338761a5fc7c9c377c318fb9e/v6.x/cjktty-6.6.patch
splitdiff -D split-v6.6 -a -p1 cjktty-6.6.patch
```

## v6.7

```
wget -c https://github.com/zhmars/cjktty-patches/raw/0d0015730edd2190dee7233f87dd72c423bb75e9/v6.x/cjktty-6.7.patch
splitdiff -D split-v6.7 -a -p1 cjktty-6.7.patch
```

## v6.8

* Add Nerd font

```
wget -c https://github.com/andot/cjktty-patches/raw/23b6c4a7753dc68f4768d27777a9b86c43ee5a94/v6.x/cjktty-6.8.patch
splitdiff -D split-v6.8 -a -p1 cjktty-6.8.patch
```

## v6.9

* 兼容反馈: https://github.com/bigshans/cjktty-patches/issues/1
    - 6.9 的补丁可以应用到 6.12

```
wget -c https://github.com/bigshans/cjktty-patches/raw/1e9bdcfe6b59efd1a45f456d0d57ca3beca13f51/v6.x/cjktty-6.9.patch
splitdiff -D split-v6.9 -a -p1 cjktty-6.9.patch
```

## v6.10

* Add Nerd font

```
wget -c https://github.com/wjz304/cjktty-patches/raw/c60c324e927794d0daa55f4b64804d5e13653e37/v6.x/cjktty-6.10.patch
splitdiff -D split-v6.10 -a -p1 cjktty-6.10.patch
```

## font patches

* Unifont 15.1

```
diff -Nu split-v6.{6,7}/lib_fonts_font_cjk_16x16.h.patch
diff -Nu split-v6.{6,9}/lib_fonts_font_cjk_16x16.h.patch
rm -v split-v6.{7,9}/lib_fonts_font_cjk_16x16.h.patch

mkdir Unifont-15.1/
mv -v split-v6.6/lib_fonts_font_cjk_16x16.h.patch Unifont-15.1/
wget -c https://github.com/zhmars/cjktty-patches/raw/b43d618da6d6536338761a5fc7c9c377c318fb9e/cjktty-add-cjk32x32-font-data.patch \
     -O Unifont-15.1/cjktty-add-cjk32x32-font-data.patch
```

* Nerd font

```
diff -Nu split-v6.{8,10}/lib_fonts_font_cjk_16x16.h.patch
rm -v split-v6.10/lib_fonts_font_cjk_16x16.h.patch

mkdir Nerd-font/
mv -v split-v6.8/lib_fonts_font_cjk_16x16.h.patch Nerd-font/
wget -c https://github.com/andot/cjktty-patches/raw/23b6c4a7753dc68f4768d27777a9b86c43ee5a94/cjktty-add-cjk32x32-font-data.patch \
     -O Nerd-font/cjktty-add-cjk32x32-font-data.patch
```

## other patches

* remove v6.10

```
diff -Nur split-v6.9/ split-v6.10/
rm -rv split-v6.10/
```

* remove v6.7

```
diff -Nur split-v6.6/ split-v6.7/
rm -rv  split-v6.7/
```

* v6.6 vs v6.8

```
diff -Nur split-v6.6/ split-v6.8/ | delta -
diff -Nu split-v6.{6,8}/drivers_tty_vt_vt.c.patch
```

Hunk: ([for Nerd font?](https://github.com/zhmars/cjktty-patches/pull/19))
```
@@ -2762,5 +3215,7 @@ static int vc_con_write_normal(struct vc_data *vc, int tc, int c,

	/* Now try to find out how to display it */
-	tc = conv_uni_to_pc(vc, tc);
+	if ((tc < 0xf000) || (tc > 0xf0ff)) {
+		tc = conv_uni_to_pc(vc, tc);
+	}
	if (tc & ~charmask) {
		if (tc == -1 || tc == -2)
```

* v6.8 vs v6.9

```
diff -Nur split-v6.8/ split-v6.9/ | delta -
diff -Nu split-v6.{6,9}/drivers_tty_vt_vt.c.patch
diff -Nu split-v6.{6,9}/drivers_video_fbdev_core_bitblit.c.patch
diff -Nu split-v6.{6,9}/drivers_video_fbdev_core_fbcon_ccw.c.patch
diff -Nu split-v6.{6,9}/drivers_video_fbdev_core_fbcon.c.patch
diff -Nu split-v6.{6,9}/drivers_video_fbdev_core_fbcon_cw.c.patch
diff -Nu split-v6.{6,9}/drivers_video_fbdev_core_fbcon_ud.c.patch
```

# Generate patches

## font header patches

* Unifont from v6.6

```
mkdir font-headers/
headersrc='Unifont-15.1/lib_fonts_font_cjk_16x16.h.patch'
headerdst='font-headers/Unifont15.1-font_cjk_16x16.h.patch'
grep -n '/font_cjk_32x32.c' $headersrc | awk -F: '{print $1}' # line 196613

cat >$headerdst <<EOF
diff --git a/lib/fonts/font_cjk_16x16.h b/lib/fonts/font_cjk_16x16.h
new file mode 100644
index 0000000..54e7e0d
EOF
sed -n '1,196612p' $headersrc >>$headerdst
gzip $headerdst

mv -v 'Unifont-15.1/cjktty-add-cjk32x32-font-data.patch' \
      'font-headers/Unifont15.1-enable-font_cjk_32x32.patch'
gzip 'font-headers/Unifont15.1-enable-font_cjk_32x32.patch'
```

* Nerd font from v6.8

```
headersrc='Nerd-font/lib_fonts_font_cjk_16x16.h.patch'
headerdst='font-headers/Nerdfont-font_cjk_16x16.h.patch'
grep -n '/font_cjk_32x32.c' $headersrc | awk -F: '{print $1}' # line 196613

cat >$headerdst <<EOF
diff --git a/lib/fonts/font_cjk_16x16.h b/lib/fonts/font_cjk_16x16.h
new file mode 100644
index 000000000..54e7e0db4
EOF
sed -n '1,196612p' $headersrc >>$headerdst
gzip $headerdst

mv -v 'Nerd-font/cjktty-add-cjk32x32-font-data.patch' \
      'font-headers/Nerdfont-enable-font_cjk_32x32.patch'
gzip 'font-headers/Nerdfont-enable-font_cjk_32x32.patch'
```

## v6.6 (LTS)

* include + lib

```
mkdir v6.6/
## patches todo
patches=(
    split-v6.6/include_linux_font.h.patch
    split-v6.6/lib_fonts_Kconfig.patch
    split-v6.6/lib_fonts_Makefile.patch
    split-v6.6/lib_fonts_font_cjk_16x16.c.patch
    #split-v6.6/lib_fonts_font_cjk_16x16.h.patch # w/o font header patches
    split-v6.6/lib_fonts_font_cjk_32x32.c.patch
    split-v6.6/lib_fonts_fonts.c.patch
)

## edit font_cjk_16x16.c.patch, font_cjk_32x32.c.patch
grep -n '/font_cjk_16x16.h' split-v6.6/lib_fonts_font_cjk_16x16.c.patch # line 31
sed -i '31,33d' split-v6.6/lib_fonts_font_cjk_16x16.c.patch
sed -i '1i diff --git a/lib/fonts/font_cjk_32x32.c b/lib/fonts/font_cjk_32x32.c\
new file mode 100644\
index 0000000..88ba3f2' split-v6.6/lib_fonts_font_cjk_32x32.c.patch

## merge
cat ${patches[@]} > v6.6/cjktty-include-lib.patch
```

* drivers

```
## patches todo
patches_a=(
    split-v6.6/drivers_tty_vt_selection.c.patch
    split-v6.6/drivers_tty_vt_vt.c.patch
)
patches_b=(
    split-v6.6/drivers_video_fbdev_core_bitblit.c.patch
    split-v6.6/drivers_video_fbdev_core_fbcon.c.patch
    split-v6.6/drivers_video_fbdev_core_fbcon.h.patch
    split-v6.6/drivers_video_fbdev_core_fbcon_ccw.c.patch
    split-v6.6/drivers_video_fbdev_core_fbcon_cw.c.patch
    split-v6.6/drivers_video_fbdev_core_fbcon_rotate.c.patch
    split-v6.6/drivers_video_fbdev_core_fbcon_ud.c.patch
)

## merge
cat ${patches_a[@]} > v6.6/cjktty-drivers-tty.patch
cat ${patches_b[@]} > v6.6/cjktty-drivers-video.patch
```

Check patches:

```
zcat font-headers/Unifont15.1-font_cjk_16x16.h.patch.gz > /tmp/cjktty-v6.6-font_cjk_16x16.h.patch
grep -n '/font_cjk_32x32.c' v6.6/cjktty-include-lib.patch # line 111
sed '110r /tmp/cjktty-v6.6-font_cjk_16x16.h.patch' v6.6/cjktty-include-lib.patch > /tmp/cjktty-v6.6-include-lib.patch
cat v6.6/cjktty-drivers-tty.patch v6.6/cjktty-drivers-video.patch  /tmp/cjktty-v6.6-include-lib.patch > /tmp/cjktty-v6.6-test.patch
diff -Nu cjktty-6.6.patch /tmp/cjktty-v6.6-test.patch
```

## v6.12 (LTS)

* include + lib

```
mkdir v6.12/
## patches todo
patches=(
    split-v6.9/include_linux_font.h.patch
    split-v6.9/lib_fonts_Kconfig.patch
    split-v6.9/lib_fonts_Makefile.patch
    split-v6.9/lib_fonts_font_cjk_16x16.c.patch
    #split-v6.9/lib_fonts_font_cjk_16x16.h.patch # w/o font header patches
    split-v6.9/lib_fonts_font_cjk_32x32.c.patch
    split-v6.9/lib_fonts_fonts.c.patch
)

## edit font_cjk_16x16.c.patch, font_cjk_32x32.c.patch
grep -n '/font_cjk_16x16.h' split-v6.9/lib_fonts_font_cjk_16x16.c.patch # line 31
sed -i '31,33d' split-v6.9/lib_fonts_font_cjk_16x16.c.patch
sed -i '1i diff --git a/lib/fonts/font_cjk_32x32.c b/lib/fonts/font_cjk_32x32.c\
new file mode 100644\
index 000000000..88ba3f278' split-v6.9/lib_fonts_font_cjk_32x32.c.patch

## merge
cat ${patches[@]} > v6.12/cjktty-include-lib.patch
```

* drivers

```
## patches todo
patches_a=(
    split-v6.9/drivers_tty_vt_selection.c.patch
    split-v6.9/drivers_tty_vt_vt.c.patch
)
patches_b=(
    split-v6.9/drivers_video_fbdev_core_bitblit.c.patch
    split-v6.9/drivers_video_fbdev_core_fbcon.c.patch
    split-v6.9/drivers_video_fbdev_core_fbcon.h.patch
    split-v6.9/drivers_video_fbdev_core_fbcon_ccw.c.patch
    split-v6.9/drivers_video_fbdev_core_fbcon_cw.c.patch
    split-v6.9/drivers_video_fbdev_core_fbcon_rotate.c.patch
    split-v6.9/drivers_video_fbdev_core_fbcon_ud.c.patch
)

## merge
cat ${patches_a[@]} > v6.12/cjktty-drivers-tty.patch
cat ${patches_b[@]} > v6.12/cjktty-drivers-video.patch
```

Check patches:

```
zcat font-headers/Unifont15.1-font_cjk_16x16.h.patch.gz > /tmp/cjktty-v6.12-font_cjk_16x16.h.patch
grep -n '/font_cjk_32x32.c' v6.12/cjktty-include-lib.patch # line 111
sed '110r /tmp/cjktty-v6.12-font_cjk_16x16.h.patch' v6.12/cjktty-include-lib.patch > /tmp/cjktty-v6.12-include-lib.patch
cat v6.12/cjktty-drivers-tty.patch v6.12/cjktty-drivers-video.patch  /tmp/cjktty-v6.12-include-lib.patch > /tmp/cjktty-v6.12-test.patch
diff -Nu cjktty-6.9.patch /tmp/cjktty-v6.12-test.patch
```

