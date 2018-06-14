``` shell
cat satoumi.txt | fstcompile --acceptor --{i,o}symbols=syms.txt | fstdraw --acceptor --portrait --{i,o}symbols=syms.txt | dot -Tsvg > fst.svg
fstcompile --acceptor --{i,o}symbols=syms.txt satoumi.txt | fstarcsort > satoumi.fst
fstcompose transliterator.fst satoumi.fst \
    | fstrmepsilon \
    | fstprint --{i,o}symbols=syms.txt \
    | sed 's/\t6/\t5/g' \
    | fstcompile --{i,o}symbols=syms.txt \
    | fstconnect \
    | fstdraw --portrait --{i,o}symbols=syms.txt \
    | dot -Tsvg > transliterate-satoumi.svg
# edit temp to merge states 5 and 6

```
