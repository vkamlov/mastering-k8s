As I decided to use Oracle ARM instance for this homework, I had to use non-standard way to use perf and flamegraph tools. Here are the steps I used to generate flamegraph:

1. Take clean ubuntu container and install required packages:
   I had a lot of problems with oracle kernel version for ubuntu, so this script was really helpful:
   https://gist.github.com/karlivory/9111f906f4eb06370b2b237c62a6b00e

2. Run perf record and generate flamegraph in the same container, on the same node, cause making graph on the local amd64 machine using perf data from remote arm machine shows unrecognized syscalls.
    I used official repo to generate flamegraph:
    https://github.com/brendangregg/FlameGraph

