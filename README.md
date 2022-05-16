# site_performance_timer

The site performance timer runs a large number of selenium jobs, simulating a specified number of users, and
generates stats for each run. Under the hood, it launches many instances of the page_performance_timer docker container
in parallel, and logs the stats (stats are in the influxdb wire protocol which can be posted directly to influx),
along with a csv file that logs error output per run.

## Output

### standard output:

The standard output is the same as what's generated by the page_performance_timer. The run_id can be used to
distinguish each unique execution.
```
user_flow_performance,server=https://usegalaxy.org.au,action=login_page_load,run_id=079473df-d440-4afa-a573-241b9d4eb1bb,end_step=tool_search_load,workflow_name=Selenium_test_1 time_taken=1.9470009803771973 1651127098165591337
user_flow_performance,server=https://usegalaxy.org.au,action=home_page_load,run_id=079473df-d440-4afa-a573-241b9d4eb1bb,end_step=tool_search_load,workflow_name=Selenium_test_1 time_taken=3.9667348861694336 1651127102132338889
user_flow_performance,server=https://usegalaxy.org.au,action=tool_search_load,run_id=079473df-d440-4afa-a573-241b9d4eb1bb,end_step=tool_search_load,workflow_name=Selenium_test_1 time_taken=3.5575649738311768 1651127105689919752
```

### log output:

The log output in csv format can be used to look at per execution runtime and any error output.

```csv
Seq,Host,Starttime,JobRuntime,Send,Receive,Exitval,Signal,Command,V1,Stdout,Stderr
2,115.146.85.100,1651127094.239,12.347,0,681,0,0,"sudo docker run -e GALAXY_USERNAME=selenium_test_user_1@mailinator.com -e GALAXY_PASSWORD=some_password -e GALAXY_SERVER=https://usegalaxy.org.au usegalaxyau/page_perf_timer:latest --end_step=tool_search_load","sudo docker run -e GALAXY_USERNAME=selenium_test_user_1@mailinator.com -e GALAXY_PASSWORD=some_password -e GALAXY_SERVER=https://usegalaxy.org.au usegalaxyau/page_perf_timer:latest --end_step=tool_search_load","user_flow_performance,server=https://usegalaxy.org.au,action=login_page_load,run_id=079473df-d440-4afa-a573-241b9d4eb1bb,end_step=tool_search_load,workflow_name=Selenium_test_1 time_taken=1.9470009803771973 1651127098165591337
user_flow_performance,server=https://usegalaxy.org.au,action=home_page_load,run_id=079473df-d440-4afa-a573-241b9d4eb1bb,end_step=tool_search_load,workflow_name=Selenium_test_1 time_taken=3.9667348861694336 1651127102132338889
user_flow_performance,server=https://usegalaxy.org.au,action=tool_search_load,run_id=079473df-d440-4afa-a573-241b9d4eb1bb,end_step=tool_search_load,workflow_name=Selenium_test_1 time_taken=3.5575649738311768 1651127105689919752
",
1,115.146.85.212,1651127094.234,36.069,0,2113,0,0,"sudo docker run -e GALAXY_USERNAME=selenium_test_user_1@mailinator.com -e GALAXY_PASSWORD=some_password -e GALAXY_SERVER=https://usegalaxy.org.au usegalaxyau/page_perf_timer:latest --end_step=workflow_run_page_load --workflow_name Selenium_test_3","sudo docker run -e GALAXY_USERNAME=selenium_test_user_1@mailinator.com -e GALAXY_PASSWORD=some_password -e GALAXY_SERVER=https://usegalaxy.org.au usegalaxyau/page_perf_timer:latest --end_step=workflow_run_page_load --workflow_name Selenium_test_3","user_flow_performance,server=https://usegalaxy.org.au,action=login_page_load,run_id=0bb2abe4-032b-45cf-bed8-875bc881b9ce,end_step=workflow_run_page_load,workflow_name=Selenium_test_3 time_taken=1.8656761646270752 1651127097921758692
user_flow_performance,server=https://usegalaxy.org.au,action=home_page_load,run_id=0bb2abe4-032b-45cf-bed8-875bc881b9ce,end_step=workflow_run_page_load,workflow_name=Selenium_test_3 time_taken=4.092638969421387 1651127102014410233
```

## Usage

1. Generate the tests

   Before running the tests, a list of tests to be executed should be generated as follows:
   ```bash
    python3 generate_tests.py -s "https://usegalaxy.org.au" -u selenium_test_user_1@mailinator.com -p "some_password" --num_tests 1000 > list_of_tests.txt
   ```

    The parameters define which server to target, and a username/password to use for running the selenium tests. The output is a simple list of commands to execute, generated randomly according to the distribution specified
    in the [acceptance test criteria document](https://docs.google.com/document/d/1uwB1EWWWFhbKdureI0Vocc5a6aP9t7UQiusntARjMeQ/edit), as follows:

    ```bash
    sudo docker run -e GALAXY_USERNAME=selenium_test_user_1@mailinator.com -e GALAXY_PASSWORD=some_password -e GALAXY_SERVER=https://usegalaxy.org.au usegalaxyau/page_perf_timer:latest --end_step=workflow_run_page_load --workflow_name Selenium_test_3
    sudo docker run -e GALAXY_USERNAME=selenium_test_user_1@mailinator.com -e GALAXY_PASSWORD=some_password -e GALAXY_SERVER=https://usegalaxy.org.au usegalaxyau/page_perf_timer:latest --end_step=tool_search_load
    sudo docker run -e GALAXY_USERNAME=selenium_test_user_1@mailinator.com -e GALAXY_PASSWORD=some_password -e GALAXY_SERVER=https://usegalaxy.org.au usegalaxyau/page_perf_timer:latest --end_step=workflow_history_summary_load --workflow_name Selenium_test_3
    ```

2. Execute the tests

    To execute the tests generated above:
    ```bash
    cat list_of_tests.txt | parallel -S 4/115.146.85.212 -S 4/115.146.85.100 -j 1 --results test_results.csv > test_results.txt
    ```

    The above will use two execution nodes (115.146.85.212 and 115.146.85.100), each with 4 cores (the prefix in front of the ip),
    and pipe the influx stats into `test_results.txt` and output log into `test_results.csv`. The `-j` parameter specifies the number
    of users to simulate in parallel.

    The execution nodes must have docker installed and have ssh access enabled, but can otherwise be vanilla ubuntu nodes.
    It must also be possible to perform passwordless authentication against the execution nodes. Configure your
    `~/.ssh/config` file as follows to do so:
    ```
    Host 115.146.85.212
    User ubuntu
    IdentityFile /home/ubuntu/keys/cloudman_keypair_rc.pem

    Host 115.146.85.100
    User ubuntu
    IdentityFile /home/ubuntu/keys/cloudman_keypair_rc.pem
    ```

### Extra commands

#### Updating the docker container
If the docker container for the page_performance_timer is updated, use the following command to update it on all nodes:
```bash
parallel --nonall -S 4/115.146.85.212 -S 4/115.146.85.100 sudo docker pull usegalaxyau/page_perf_timer:latest
```

#### Increasing shh connections
```bash
parallel --nonall -S 115.146.86.82 -S 115.146.84.23 -S 115.146.86.236 -S 115.146.86.71 -S 115.146.84.220 -S 115.146.84.115 'sudo grep -q "^MaxSessions" /etc/ssh/sshd_config && sudo sed "s/^MaxSessions.*/MaxSessions 50/" -i /etc/ssh/sshd_config || sudo sed "$ a\MaxSessions 50" -i /etc/ssh/sshd_config'
parallel --nonall -S 115.146.86.82 -S 115.146.84.23 -S 115.146.86.236 -S 115.146.86.71 -S 115.146.84.220 -S 115.146.84.115 'sudo grep -q "^MaxStartups" /etc/ssh/sshd_config && sudo sed "s/^MaxStartups.*/MaxStartups 100:30:1000/" -i /etc/ssh/sshd_config || sudo sed "$ a\MaxStartups MaxStartups 100:30:1000" -i /etc/ssh/sshd_config'
parallel --nonall -S 115.146.86.82 -S 115.146.84.23 -S 115.146.86.236 -S 115.146.86.71 -S 115.146.84.220 -S 115.146.84.115 sudo service ssh restart
```
