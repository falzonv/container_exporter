# container_exporter

This script exposes the CPU% and MEM% metrics (from the `docker stats` command) of all running Docker containers to a [Prometheus](https://prometheus.io) collector.
For this, the script collects the metrics and arrange them in the format expected by Prometheus.
Then, thanks to the TCP handler [`xinetd`](https://linux.die.net/man/8/xinetd), when Prometheus will connect to the specified port, the script will start and send him an HTTP result displaying the collected metrics (see example lower).

Of course other very advanced solutions exist to monitor all the aspects of Docker containers.
But if you only really need the CPU and memory usage, this script might be a lighter choice.
It will provide both information as percentages and, if you know the size of the installed memory, you can easily retrieve the amount of memory used (see formula lower).

This project is licensed under the terms of the [GNU General Public License v3.0](https://www.gnu.org/licenses/gpl-3.0.txt).

## Suggested installation process

### On the server running the Docker containers

Install the package `xinetd` with your favorite package manager (for example `sudo apt install xinetd`).

Then, from the folder of your choice, download and extract the ZIP archive from GitHub :

```
wget "https://github.com/falzonv/container_exporter/archive/main.zip"
unzip main.zip
```

***Note :*** *if you are familiar with Git, you may prefer to use `git clone "https://github.com/falzonv/container_exporter" container_exporter-main` so you can easily get updates with `git pull`*

Copy the file `service_container_exporter` in the folder `/etc/xinetd.d` and edit this file with your favorite editor :

```
sudo cp container_exporter-main/service_container_exporter /etc/xinetd.d
sudo vi /etc/xinetd.d/service_container_exporter
```

***Note :*** *using the `git clone` alternative, you may prefer to create a symbolic link using `ln -s /path/to/folder/container_exporter-main/service_container_export /etc/xinetd.d/service_container_exporter`*

The only settings you need to configure in the `service_container_exporter` file are the exact location of the `container_exporter.sh` file (see example in the file) and the port that Prometheus will use to collect the metrics (by default it is set to 9117).

Finally, you can test that everything is working by running `nc localhost <chosen_port>` :

```
user@hostname:~$ nc localhost 9117
HTTP/1.1 200 OK
Date: mer. 20 janv. 2021 21:35:13 CET
Content-Length: 778
Content-Type: text/plain
Connection: close

# HELP container_cpu_percent The CPU% value from docker-stats.
# TYPE container_cpu_percent gauge
container_cpu_percent{container="samba"} 0.01
container_cpu_percent{container="webserv"} 0.01
container_cpu_percent{container="grafana"} 0.05
container_cpu_percent{container="prometheus"} 0.17
container_cpu_percent{container="exportsnmp"} 0.00
container_cpu_percent{container="pihole"} 0.16
# HELP container_mem_percent The MEM% value from docker-stats.
# TYPE container_mem_percent gauge
container_mem_percent{container="samba"} 0.11
container_mem_percent{container="webserv"} 0.09
container_mem_percent{container="grafana"} 0.36
container_mem_percent{container="prometheus"} 1.17
container_mem_percent{container="exportsnmp"} 0.13
container_mem_percent{container="pihole"} 0.23

user@hostname:~$
```

Note that it will take a few seconds for the `docker stats` command to complete before anything is displayed (I guess the amount of time is relative to the number of containers, for the example above it took between 2 and 3 seconds).
You may also notice that `nc` does not display the prompt back after the script has finished, just hit the `Enter` key and the prompt will appear.

### On the server running Prometheus

It may be the same server than the one running the Docker container that you configured above (as a matter of fact, you can see in the list of metrics above that Prometheus may even be himself one of the monitored Docker containers).

The configuration is pretty simple as you only need to add the following lines in the `scrape_configs:` section of your `/etc/prometheus/prometheus.yml` file :

```
scrape_configs:

  # Collect metrics about the Docker containers
  - job_name : 'getContainerStats'
    metrics_path: /
    static_configs:
      - targets:
        - 192.168.10.10:9117
```

The IP address should belong to the server where the `container_exporter` has been installed and the port number should be consistent with what has been set in the `service_container_exporter` file.

After this, restart Prometheus for him to take the new settings in account and you shoud you quickly retrieve the metrics in your interface, depending on your polling interval (the default is one minute).

## Usage

If you are already using Prometheus and Grafana (and if you have read that far !), I am sure you already know how to use these newly collected metrics.
However, here are some useful information to make things even easier :

  * The collected metrics are named `container_cpu_percent` and `container_mem_percent`
  * When the metrics are registered, the name of the container is associated with them. Therefore, if you want the CPU usage from the `pihole` container, in Grafana for example, simply use `container_cpu_percent{container="pihole"}`
  * You can use the `sum()` function to get the total used percentage, for example `sum(container_cpu_percent)`
  * Finally, you can easily compute the real memory usage as described below

### Get the real memory usage in bytes

As you probably already know, the `docker stats` command already displays the memory usage but in a way which is not very easy to collect due to the units.
Here is an example with only the relevant columns :

```
user@hostname:~$ docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemPerc}}\t{{.MemUsage}}"
NAME                CPU %               MEM %               MEM USAGE / LIMIT
samba               0.01%               0.11%               7.988MiB / 7.088GiB
webserv             0.01%               0.09%               6.531MiB / 7.088GiB
grafana             0.05%               0.36%               25.84MiB / 7.088GiB
prometheus          0.17%               1.17%               85.25MiB / 7.088GiB
exportsnmp          0.00%               0.13%               9.43MiB / 7.088GiB
pihole              0.16%               0.23%               16.96MiB / 7.088GiB
user@hostname:~$
```

With the `container_exporter`, you are already collecting the value of MEM% for each container and, as it is a percentage, it is quite easy to compute the real usage which is displayed by `docker stats` (you can do this in a Grafana query for example).

Here is the global formula : `container_mem_percent{container="<name_of_container>"} * <amount_of_memory> * 10`

The amount of memory can be found in the column `LIMIT` from `docker stats` (in the example above, it is 7.088) and will probably never change (unless you change the hardware or you configure a memory limit for a specific container).

