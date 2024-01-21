## Localnet

1. Launch the localnet. Quick script to do this

        make build
        build/simd init keruch --chain-id my-test-chain
        build/simd keys add my_validator --keyring-backend test
        build/simd genesis add-genesis-account my_validator 100000000000stake
        build/simd genesis gentx my_validator 100000000stake --chain-id my-test-chain --keyring-backend test
        build/simd genesis collect-gentxs
        build/simd start

1. I added a GenesisTime query for x/mint to get the genesis block time. You can check the genesis time by executing

        build/simd query mint genesis-time

   In my case the result is

         genesis_time: "2024-01-21T11:04:41.307008Z"

1. Just for the testing purposes I changed the inflation schedule from annual (as in the ADR) to per minute. The initial
   values are the same as in the ADR, so we expect the results below

   | Minute after the genesis | Inflation (%)     |
      |--------------------------|-------------------|
   | 0                        | 8.00              |
   | 1                        | 7.20              |
   | 2                        | 6.48              |
   | 3                        | 5.832             |
   | 4                        | 5.2488            |
   | 5                        | 4.72392           |
   | 6                        | 4.251528          |
   | 7                        | 3.8263752         |
   | 8                        | 3.44373768        |
   | 9                        | 3.099363912       |
   | 10                       | 2.7894275208      |
   | 11                       | 2.51048476872     |
   | 12                       | 2.259436291848    |
   | 13                       | 2.0334926626632   |
   | 14                       | 1.83014339639688  |
   | 15                       | 1.647129056757192 |
   | 16                       | 1.50              |
   | 17                       | 1.50              |

1. For convenience, I added more attributes into the block result events, so you can check all the inflation param there.
   The 1st block is the genesis block, so there is no useful info. Therefore, let's start with the 2nd block

   ```bash
   build/simd query block-results 2 --output json | jq '.finalize_block_events | map(select(.type | contains("mint"))) | first'
   ```

   ```json
   {
    "type": "mint",
    "attributes": [
      {
        "key": "inflation",
        "value": "0.080000000000000000",
        "index": true
      },
      {
        "key": "annual_provisions",
        "value": "8000000000.000000000000000000",
        "index": true
      },
      {
        "key": "previous_block_time",
        "value": "2024-01-21 11:04:41.307008 +0000 UTC",
        "index": true
      },
      {
        "key": "current_block_time",
        "value": "2024-01-21 11:04:47.508957 +0000 UTC",
        "index": true
      },
      {
        "key": "block_provision",
        "value": "826926533",
        "index": true
      },
      {
        "key": "mode",
        "value": "BeginBlock",
        "index": true
      }
    ]
   }
   ```
   
   Here we can see:
   * current inflation, which is 8% since it's the 0 minute
   * initial annual_provisions
   * previous_block_time, which equals to the genesis time
   * current_block_time
   * block_provision, which is calculated based on the diff between current_block_time and previous_block_time

1. Let's run this query for further blocks and see the results

   ```bash
   build/simd query block-results 3 --output json | jq '.finalize_block_events | map(select(.type | contains("mint"))) | first'
   ```

   ```json
   {
    "type": "mint",
    "attributes": [
      {
        "key": "inflation",
        "value": "0.080000000000000000",
        "index": true
      },
      {
        "key": "annual_provisions",
        "value": "8000000000.000000000000000000",
        "index": true
      },
      {
        "key": "previous_block_time",
        "value": "2024-01-21 11:04:47.508957 +0000 UTC",
        "index": true
      },
      {
        "key": "current_block_time",
        "value": "2024-01-21 11:04:52.526228 +0000 UTC",
        "index": true
      },
      {
        "key": "block_provision",
        "value": "668969466",
        "index": true
      },
      {
        "key": "mode",
        "value": "BeginBlock",
        "index": true
      }
    ]
   }
   ```
   
   The 3rd block has the same inflation as the 2nd. But here we can see that block_provision differs from the 2nd block since the time diff between blocks is less.

   * 1-2 diff = 6 secs 20 ms, provision = 826926533
   * 2-3 diff = 5 secs 02 ms, provision = 668969466

1. Now let's try querying the next minute's block. For example, the 13th block was executed after more than a minute after the genesis

   ```bash
   build/simd query block-results 13 --output json | jq '.finalize_block_events | map(select(.type | contains("mint"))) | first'
   ```

   ```json
   {
    "type": "mint",
    "attributes": [
      {
        "key": "inflation",
        "value": "0.072000000000000000",
        "index": true
      },
      {
        "key": "annual_provisions",
        "value": "7200000000.000000000000000000",
        "index": true
      },
      {
        "key": "previous_block_time",
        "value": "2024-01-21 11:05:37.692178 +0000 UTC",
        "index": true
      },
      {
        "key": "current_block_time",
        "value": "2024-01-21 11:05:42.711035 +0000 UTC",
        "index": true
      },
      {
        "key": "block_provision",
        "value": "602262840",
        "index": true
      },
      {
        "key": "mode",
        "value": "BeginBlock",
        "index": true
      }
    ]
   }
   ```
   
   As we can see, the inflation here is decreased according to the table above.

1. Let's repeat it for the further blocks with more than 1 minute diff. Here I printed only inflation and current_block_time for convenience.

   Block 25:

   ```bash
   build/simd query block-results 25 --output json | jq '.finalize_block_events | map(select(.type | contains("mint"))) | first' | jq '.attributes | map(select(.key == "inflation" or .key == "current_block_time"))'
   ```

   ```json
   [
     {
       "key": "inflation",
       "value": "0.064800000000000000",
       "index": true
     },
     {
       "key": "current_block_time",
       "value": "2024-01-21 11:06:42.91253 +0000 UTC",
       "index": true
     }
   ]
   ```

   Block 37:

   ```bash
   build/simd query block-results 37 --output json | jq '.finalize_block_events | map(select(.type | contains("mint"))) | first' | jq '.attributes | map(select(.key == "inflation" or .key == "current_block_time"))'
   ```

   ```json
   [
     {
       "key": "inflation",
       "value": "0.058320000000000000",
       "index": true
     },
     {
       "key": "current_block_time",
       "value": "2024-01-21 11:07:43.128997 +0000 UTC",
       "index": true
     }
   ]
   ```

   Block 49:

   ```bash
   build/simd query block-results 49 --output json | jq '.finalize_block_events | map(select(.type | contains("mint"))) | first' | jq '.attributes | map(select(.key == "inflation" or .key == "current_block_time"))'
   ```

   ```json
   [
     {
       "key": "inflation",
       "value": "0.052488000000000000",
       "index": true
     },
     {
       "key": "current_block_time",
       "value": "2024-01-21 11:08:43.349457 +0000 UTC",
       "index": true
     }
   ]
   ```

   Block 61:

   ```bash
   build/simd query block-results 61 --output json | jq '.finalize_block_events | map(select(.type | contains("mint"))) | first' | jq '.attributes | map(select(.key == "inflation" or .key == "current_block_time"))'
   ```

   ```json
   [
     {
       "key": "inflation",
       "value": "0.047239200000000000",
       "index": true
     },
     {
       "key": "current_block_time",
       "value": "2024-01-21 11:09:43.567091 +0000 UTC",
       "index": true
     }
   ]
   ```

   Block 73:

   ```bash
   build/simd query block-results 73 --output json | jq '.finalize_block_events | map(select(.type | contains("mint"))) | first' | jq '.attributes | map(select(.key == "inflation" or .key == "current_block_time"))'
   ```

   ```json
   [
     {
       "key": "inflation",
       "value": "0.042515280000000000",
       "index": true
     },
     {
       "key": "current_block_time",
       "value": "2024-01-21 11:10:43.776231 +0000 UTC",
       "index": true
     }
   ]
   ```
   
   Let's stop there. As we can see, the values are the same as in the table above. Finally, check the block where the inflation reaches its minimum.

   Block 193 (16 mins from the genesis)

   ```bash
   build/simd query block-results 193 --output json | jq '.finalize_block_events | map(select(.type | contains("mint"))) | first' | jq '.attributes | map(select(.key == "inflation" or .key == "current_block_time"))'
   ```

   ```json
   [
     {
       "key": "inflation",
       "value": "0.015000000000000000",
       "index": true
     },
     {
       "key": "current_block_time",
       "value": "2024-01-21 11:20:45.929289 +0000 UTC",
       "index": true
     }
   ]
   ```
   
   All further blocks will have the same inflation. For recon, block 250 (25 mins from the genesis)

   ```bash
   build/simd query block-results 250 --output json | jq '.finalize_block_events | map(select(.type | contains("mint"))) | first' | jq '.attributes | map(select(.key == "inflation" or .key == "current_block_time"))'
   ```

   ```json
   [
     {
       "key": "inflation",
       "value": "0.015000000000000000",
       "index": true
     },
     {
       "key": "current_block_time",
       "value": "2024-01-21 11:29:27.419997 +0000 UTC",
       "index": true
     }
   ]
   ```
