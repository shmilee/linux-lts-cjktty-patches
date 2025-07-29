# Generate patches

* vim
* patch
* diffutils
* patchutils

# Split example patches

## v6.6 (LTS)

```
wget -c https://github.com/zhmars/cjktty-patches/raw/b43d618da6d6536338761a5fc7c9c377c318fb9e/v6.x/cjktty-6.6.patch
splitdiff -D split-v6.6 -a -p1 cjktty-6.6.patch
```

## v6.8

* Add Nerd font

```
wget -c https://github.com/andot/cjktty-patches/raw/23b6c4a7753dc68f4768d27777a9b86c43ee5a94/v6.x/cjktty-6.8.patch
splitdiff -D split-v6.8 -a -p1 cjktty-6.8.patch
```

## v6.9

* 兼容反馈: https://github.com/bigshans/cjktty-patches/issues/1

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
