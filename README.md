This simulator is built upon [AMLSim](https://github.com/IBM/AMLSim). Please first have a look at that for installing dependencies and general introduction. This README focuses more on detailed configuration and description.

I was looking for synthetic transaction datasets to do Anti-Money Laundering (AML) research, and realised there weren't many of them. AMLSim was the only data generator found. But the quality of the data AMLSim generated is not sufficient for scientific research. Data quality analysis is presented here: [placeholder]

The improvements made are summarised in this description.

## How to generate data?

After checking the configuration file `config.json`, please follow the instructions in AMLSim (three mandatory steps):

1. Run the commands to generate a transaction graph

    ```bash
    cd /path/to/AMLSim-R
    python3 scripts/transaction_graph_generator.py conf.json
    ```

2. Run the commands to run the transaction simulator

    ```bash
    sh scripts/build_AMLSim.sh
    sh scripts/run_AMLSim.sh conf.json
    ```

3. Run the command to convert the transaction log file to a formatted transaction `.csv` file.

    ```bash
    python3 scripts/convert_logs.py conf.json
    ```

The above sequential steps are put together in one script `scripts/run_batch.sh`, so one command should be enough to generate a dataset:

```bash
sh scripts/run_batch.sh
```

Make sure you have run `sh scripts/build_AMLSim.sh` to compile the Java files, this is a one-time work unless you have made modifications in the Java files.

## How to configure AMLSim-R?

For an AML dataset (AML stands for Anti-Money Laundering here, not Automated Machine Learning. Repeat, AML stands for Anti-Money Laundering.), the most interesting properties would be:

1. The numbers of all accounts and all transactions
2. The numbers of suspicious accounts and transactions
3. The distributions of the amount of money in normal transactions and suspicious transactions

### The numbers of all accounts and all transactions

**Accounts**

The configuration file `config.json` specifies a parameter directory:

```text
{
//...
  "input": {
    "directory": "paramFiles/100K",  // Parameter directory
    //...
  },
//...
}
```

In file `paramFIles/100K/accounts.csv`, you can specify the number of accounts with certain attributes in each line. The total number of accounts is the sum of the values in the `count` column. The following is an example configuration.

| count| min_balance| max_balance| country| buisness_type | model  | bank_id  |
| ------------- |:-------------:| :-----:| :-----:|:-----:|:-----:|:-----:|
| 5000    | 1000 | 100000 | NL | I | 1 | bank |
| 3000    | 1000 | 100000 | NL | I | 2 | bank |
| ...     | ...| ...| ...| ...| ...| ...|

**Transactions:**

The configuration file `config.json` specifies the total number simulation steps:

```text
{
  "general": {
    //...
    "total_steps": 720,  // the number of simulation steps
    //...
  },
//...
}
```

In each step, the transaction simulator (Java part) simulates a number of transactions. Changing the number of steps results in different numbers of transactions.

### The numbers of suspicious accounts and transactions

File `paramFIles/100K/alertPatterns.csv` specifies different types of suspicious (alert) patterns. Within each pattern, the range of the number of involved accounts is specified. Once the number of suspicious accounts is randomly selected from the range, the number of suspicious transactions is then also determined. For example, if 100 suspicious accounts are in a *fan_in* patter, then there would be 99 suspicious transactions, because of the nature of the pattern.

The following is an example.

| count| type | schedule_id| min_accounts| max_accounts| min_amount| max_amount| min_period| max_period | bank_id  | is_sar|
| ------------- |:-------------:| :-----:| :-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
|25|fan_in|2|5|15|100.0|10000.0|5|20|bank|True|
|35|fan_out|2|5|15|50000.0|100000.0|5|20|bank|True|
| ...    | ...| ...| ...| ...| ...| ...|...|...|...|...|

### The distributions of the amount of money

**Normal transactions**

The amount of money in every normal transaction is randomly selected from a shared range, which is specified in `config.json`:

```text
{
//...
  "default": {
    "min_amount": 1,
    "max_amount": 5000000,
    //...
  },
//...
}
```

The original AMLSim select values from this range randomly and uniformly, which is rather unrealistic. The distribution can be modified in file `src/amlsim/SimProperties.java`,line 96, function `getNormalBaseTxAmount()`.

AMLSim-R uses a power law distribution:

<img src="https://render.githubusercontent.com/render/math?math=x^{\prime} = T_{\text{min}} %2B (1 - x^{\frac{1}{11}}) \cdot (T_{\text{max}} - T_{\text{min}})">

T_min and T_max are the boundaries set above.


**Suspicious transactions**

This is also configured in file `paramFIles/100K/alertPatterns.csv`. The file specifies the number of suspicious patterns and the range of the amount of money involved. Tuning these two numbers can produce desired distributions.

The following is an example of realistic suspicious transactions, which is summarised from the annual reports from 2017 to 2019 published by Dutch Financial Intelligence Unit. See more in the Data Quality Analysis: [placeholder]

| Amount|pct_num|pct_amt|
| ------------- |:-------------:| :-----:|
|€0-€10K|79%|3%|
|€10K-€100K|16%|17%|
|€100K-€1M|5%|36%|
|€1M-€5M|1%|44%|
|Total|100%|100%|

## Miscellaneous Improvements

1. AMLSim generates edges according to a degree distribution configuration `paramFIles/100K/degree.csv`. But it was implemented in such a way that only accounts have high in-degrees can have high out-degrees. This binding is not reasonable. For example, an employee only receives money from her company (in-degree of 1) but pay multiple bills (high out-degree). An improved implementation is done in file `scripts/transaction_graph_generator.py`, line 96.
2. AMLSim provides an optional step to plot in/out-degree distributions for the transaction graph; but its output is misleading. Its output is the degree distributions for the *connection graph*, not the *transaction graph*. A connection graph is a prerequisite of a transaction graph: transactions are derived from connections. AMLSim-R outputs distributions of both graphs. The implementation is done in file `scripts/visualize/plot_distributions.py`, line 180.
