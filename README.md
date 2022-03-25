# cosmoscartography
Mapping the Cosmos


## Data Collection
Here's my script to map movement of funds through `cosmos-sdk/MsgSend`

```bash
#!/bin/bash

DAEMON_NAME="gaiacli"
RANGE_START=$1
RANGE_END=$(( RANGE_START + 100000 ))

for (( b=$RANGE_START; b<$RANGE_END; b++ )); do
        BLOCK="$(${DAEMON_NAME} q block $b --trust-node)"
        NUM_TXS="$(echo $BLOCK | jq -r '.block.data.txs | length')"
        if [[ $NUM_TXS > 0 ]]; then
                TX_HASHES="$(for i in $(echo "${BLOCK}" | jq -r .block.data.txs[]); do echo -n $i | base64 -d | sha256sum -b - | cut -d ' ' -f 1; done)"
                for TXN in $TX_HASHES; do
                        MSGS="$(${DAEMON_NAME} q tx $TXN --output json --trust-node | jq -r '.tx | select(.type=="cosmos-sdk/StdTx") | .value')"
                        NUM_MSGS="$(echo $MSGS | jq -r '.msg | length')"
                        if [[ $NUM_MSGS > 0 ]]; then
                                for (( m=0; m<$NUM_MSGS; m++ )); do
                                        echo $MSGS | jq -r " .msg[$m] | select( .type==\"cosmos-sdk/MsgSend\" ) | [ $b, .value.from_address, .value.to_address, .value.amount[0].amount ] | @csv" | tee -a cosmoshub-3_$RANGE_START.csv
                                done
                        fi
                done
        fi
done
```

This is super slow.. on order of 10,000 blocks per second.. and I estimated it to take roughly 9 days to complete.  Jack Zampolin, from Strangelove Ventures, decided to give the machine 4x resources.   Unfortunately, the search became even slower when I realized I was fighting with good ol' linux `ulimit`.

```
ERROR: Block: response error: RPC error -32603 - Internal error: open /home/user/.gaiad/data/blockstore.db/1220877.ldb: too many open files
```

I then opened up the `ulimit` and realized 9 days is far far too long.  So I decided to fork off four individual instances, since the data is largely independent and I can rapidly merge them together.

```bash
seq 2092000 100000 5200790 | xargs -n 1 -P 4 -I {} ./map_cosmos_hub3.sh {}
```

To go faster I'll be spinning up another cosmoshub-3 node, on a very fast disk/cpu machine somewhere in Helsinki

<img width="345" alt="image" src="https://user-images.githubusercontent.com/759747/158024843-3c708c9b-54d3-479b-8083-e9e77edb122b.png">

Update... well hetzner or aws didn't like me attempting to fetch 10 shards at a time (shocker).  Guess I'll go at this the old fashioned way... throuhg a small straw:

<img width="207" alt="image" src="https://user-images.githubusercontent.com/759747/158025466-23eda7b3-278d-444e-8f6a-e846d4cdf10f.png">


So now I am producing individual dumps in 100K increments.. but how will I merge?  I can simply append each file to one another and produce a very large CSV file.  


That will produce a giant view of the entirety of cosmoshub-3 transfer activity. I'll need to next figure out how to filter it to just the addresses we want to see.

## Learn
The data is still coming together, and I may need to fork this out further to rapidly gather it because time is of the essence now.

Once we have the picture we can begin to intuitively understand the flow of funds and perhaps identify other instances that shoudl be scrutinized.
