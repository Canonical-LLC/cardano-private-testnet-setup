# Run scripts to start private testnet

**Now, we get to the fun part!**  We are ready to create & start a private Cardano testnet.
The testnet will run three block producer nodes. 
After starting the testnet and running all protocol updates, we set up a PostgreSQL database for cardano-db-sync.
Then, we connect the db-sync process to one of the cardano nodes, so that
it can synchronize to the activity occurring on our testnet and write data to its database.

**Tip**: The instructions below require opening several terminal- windows or tabs.  You might prefer creating a [`tmux`](https://en.wikipedia.org/wiki/Tmux) session
and create several window panes in your session to make it easier to see everything in the same terminal window.

#### Assumptions
- This guide assumes you are running a recent version of linux.
  Specifically, these directions apply to Ubuntu (Debian). If you are using a different linux variant, please adjust as needed
- Before using this guide, you should have completed the [Install executables guide](./1-INSTALL_EXECUTABLES.md) and
  [Install PostgreSQL guide](2-INSTALL_POSTGRESQL.md).
  
## 1. Clone this project

  ```shell
  # navigate to working source directory
  cd $HOME/src
  git clone https://github.com/woofpool/cardano-private-testnet-setup
  ```

## 2. Bootstrap the network and apply protocol updates

At the time of this writing, the current era is `alonzo` and major protocol version `6`.
After we bootstrap the network in the `Byron` era/major protocol version `0`, the update scripts are applied 
one by one to advance the network into the current era and major protocol version.
In order to advance the protocol version, it requires two transactions-- 1) update proposal transaction and a 2) vote transaction.
After a protocol update has been staged, we need to wait an epoch
before the protocol version is bumped in the protocol parameters. It takes around 3 to 5 minutes for an epoch to complete based
on the testnet environment configuration.

Whether you run the process manually or use the automated script, it will take about 20-25 minutes to apply all the updates.
By running the process manually, you get to experience the update process in a similar way to how things happen in mainnet Cardano.
On the other hand, you can also run the `automate.sh` script to process things without needing to babysit the process.

Once you have completed this step, the `private-testnet` folder contains the persisted blockchain data,
which means you can restart the nodes whenever you like, and your private testnet will be running in the current era and protocol.

#### Directions
- Adjust the [config.cfg file](./scripts/config.cfg) as desired. By default, the ROOT directory is set to `private-network`
- Choose **Option 1** to run the scripts manually or choose **Option 2** to run a single script that automates the whole process
  #### Option 1: Run sequence of scripts manually

    - In **terminal #1**, run the `mkfiles` script to set up the network files
      ```shell
      # navigate to project root folder
      cd $HOME/src/cardano-private-testnet-setup
      
      # run script file (note: it's important to run scripts from the root of the project as script paths are relative to the root)
      ./scripts/mkfiles.sh
      
      # inspect script output
      # If you see "Default yaml configuration applied." at the bottom, it was successful.
      ```
    - In the same **terminal #1**, start up the all three nodes using the script that gets generated by running make-network-files
      ```shell
      # for convenience, lets create a variable to store the name of the directory for our private testnet
      # this directory was generated by running `./scripts/mkfiles.sh` above
      ROOT=private-testnet
        
      # run script to start all three nodes.
      ./$ROOT/run/all.sh
      
      # The output may show some Forge errors, but should go away once all the nodes are
      # synched to each other  
      ```

   - Apply updates to the network to advance the network protocol to latest era and protocol version
    
        - Open **terminal #2** and run the `update-1` script
          ```shell
          # navigate to project root folder
          cd $HOME/src/cardano-private-testnet-setup
        
          # set variables
          ROOT=private-testnet
          export CARDANO_NODE_SOCKET_PATH=$ROOT/node-bft1/node.sock
          
          # run script file
          ./scripts/update-1.sh
          
          # if you see an error about `<socket> does not exist`, wait a little longer and try submitting the `scripts/update-1.sh` script again
          ```
        - In the same **terminal #2**, wait for at least the next epoch to make sure the update to protocol V1 is completed.
          ```shell
          
          # run query to find out the current epoch
          cardano-cli query tip --testnet-magic 42
          
          # the query should return something like this 
          {
            "epoch": 2,
            "hash": "587799cb34f2c68a04c29204e120418351f4b449aa5286c9b4ac3244f30a7933",
            "slot": 200,
            "block": 197,
            "era": "Byron",
            "syncProgress": "100.00"
          }    
          ```
        - In **terminal 2**, run the `update-2` script  
          ```shell
          ./scripts/update-2.sh
          
          # verify the transactions submitted successfully
          # If you run the update script too early, you will see error regarding update proposal
          # Just continue waiting until the next epoch is reached and try running the update again
          ```
        - Switch to **terminal 1** and restart the nodes
          ```shell
          # stop the script process
          ctrl+c, ctrl+c  
          # and run the script again to start up the nodes
          ./$ROOT/run/all.sh  
          ```
        - In **terminal 2**, wait for a new epoch to make sure the update to protocol V2 is completed.
          ```shell
          # run query to find out the current epoch
          cardano-cli query tip --testnet-magic 42
          
          # you should see something like this
          {
            "epoch": 3,
            "hash": "d485e25f1287572ae75dbd35727ecd792595d88b66ac6e2144cbdf6dd1dab200",
            "slot": 302,
            "block": 290,
            "era": "Byron",
            "syncProgress": "100.00"
          }
          ```
        - In **terminal 2**, run the v3 update script
          ```shell  
          ./scripts/update-3.sh <current_epoch>
          
          # verify the script update ran successfully
          # if you see something like the following, it means you need to wait for another epoch
          Command failed: transaction submit  Error: Error while submitting tx: ShelleyTxValidationError ShelleyBasedEraShelley (ApplyTxError [UtxowFailure (UtxoFailure (UpdateFailure (PPUpdateWrongEpoch (EpochNo 3) (EpochNo 3) VoteForNextEpoch)))])    
          ```
        - Switch to **terminal 1** and restart the nodes
          ```shell
          # stop the script process
          ctrl+c, ctrl+c
          
          # run the script again to start up the nodes
          ./$ROOT/run/all.sh  
          ```
        - In **terminal 2**, wait for at least the next epoch to make sure the update to protocol V3 is completed.
          ```shell
          # run query to find out the current epoch
          cardano-cli query tip --testnet-magic 42
          
          # Starting in the Shelley era, we can also run query to get network protocol info
          cardano-cli query protocol-parameters --testnet-magic 42
          ```
        - Repeat the same process as you did for `update-3` for each of `update-4`, `update-5`, and `update-6`
        to advance the protocol updates to Alonzo era and protocol V6. These are the current era and protocol
        at the time of this writing.
  #### Option 2: Run automated script
    - In **terminal #1**, run the automate script. This script simply automates the process to run the sequence of steps described in Option 1.
      It takes about 20-25 minutes to complete.
      ```shell
      # navigate to project root folder
      cd $HOME/src/cardano-private-testnet-setup
      ./scripts/automate.sh
      
      # if you receive an error about "running nodes found", you will need to kill cardano node processes
      # and run the automate.sh script again
      
      # when the script completes successfully, it will show the current era and major protocol version
      # the last command that runs in the script is `wait`
      # If you kill the script, that will also kill the running nodes.
      ```
    - In **terminal #2**, you can tail the log to monitor the activity occurring in the nodes
      ```shell
      # navigate to project root folder
      cd $HOME/src/cardano-private-testnet-setup
      tail -f logs/privatenet.log
      ```
## 3. Verify the network state and wallet UTxO balances of user1

- After completing the full set of updates, verify the era, major protocol version, and utxo balances of user1 address
  ```shell
  # if necessary, set variables
  ROOT=private-testnet
  export CARDANO_NODE_SOCKET_PATH=$ROOT/node-bft1/node.sock
  
  cardano-cli query tip --testnet-magic 42 | jq '.era'
  #output
  "Alonzo"
  
  cardano-cli query protocol-parameters --testnet-magic 42 | jq '.protocolVersion.major'
  #output
  6
  
  cardano-cli query utxo --address $(cat private-testnet/addresses/user1.addr) --testnet-magic 42
  #output
                            TxHash                                 TxIx        Amount
  --------------------------------------------------------------------------------------
  b0f91ee59eb208284467b1dec0adfa8c57eb1cf7587fb7eb0599e2b8c8e885c9     0        500000000 lovelace + TxOutDatumHashNone
  b0f91ee59eb208284467b1dec0adfa8c57eb1cf7587fb7eb0599e2b8c8e885c9     1        500000000 lovelace + TxOutDatumHashNone
  ``` 
- **Troubleshooting**: If you run `cardano-cli query tip` and the blocks are not advancing or the syncProgress percent is decreasing,
  it may mean the processes running the nodes are running into garbage collection/memory issues. The author is still researching
  the cause of this issue.  In any event, the best remedy is killing the run node processes, deleting the `private-testnet` folder
  and starting over.  This garbage collection issue normally happens early in the update process.
    - If you use Ctrl+c in the window running the `run/all.sh` script, it should kill the processes that started in the background.
      Another approach is to kill the processes directly by doing:
      ```shell
      for PID in `ps -ef | grep 'cardano-node' | grep -v grep |  awk '{print $2}'`;do kill -TERM $PID; done      
      ```
    
---

Continue to next guide: [4. Attach db-sync](4-ATTACH_DB_SYNC.md)