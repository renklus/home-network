select biggest possible zip file

./immich-go upload from-google-photos --server=https://immich.ingress.prod.k8s.renklus.ch --api-key=<API-KEY> --admin-api-key=<API-KEY> --concurrent-tasks=8 --client-timeout=60m --pause-immich-jobs=true --on-errors=stop --tag="source/GoogleTakeout2026" --manage-raw-jpeg=KeepJPG --manage-burst=NoStack --log-file ./immich-go-full.log ./takeout-20261006T201224Z-1-001.zip 

rerun until no upload failure

check upload log
cat ./immich-go.log | grep -v "discarded not selected" | grep -E ' ERR | WRN '