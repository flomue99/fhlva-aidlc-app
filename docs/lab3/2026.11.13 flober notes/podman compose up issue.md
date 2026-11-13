## podman compose up issue


### needed to delete some old files

[root@37ddbed58f8b sre3_g2_beer]# podman compose up
Error: current system boot ID differs from cached boot ID; an unhandled reboot has occurred. Please delete directories "/run/containers/storage" and "/run/libpod" and re-run Podman


### delete old files

rm -rf /run/libpod /run/containers/storage /run/containers/networks


### now docker compose up works

[root@37ddbed58f8b sre3_g2_beer]# podman compose up
>>>> Executing external compose provider "/usr/bin/podman-compose". Please see podman-compose(1) for how to disable this message. <<<<

3d6cab9c2941df2441024a2f02a65ebe1b8916c3cbe7b9527f62627accea8004
...
