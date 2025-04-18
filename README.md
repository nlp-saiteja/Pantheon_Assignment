# Instructions for Reproducing Pantheon Experiments
This document provides step-by-step instructions for reproducing the experiments conducted in this study. The following instructions assume you are running Ubuntu 24.04 or a compatible Linux distribution.

## System Requirements

Ubuntu 24.04 or compatible Linux distribution
Python 2.7
Git
At least 30GB of disk space

## Setup and Installation
## 1. Install Required Dependencies
 #### Update package lists
sudo apt-get update

## Install essential build tools and libraries
sudo apt-get install -y build-essential git automake libtool libtbb-dev \
                      pkg-config iptables debhelper autotools-dev \
                      dh-autoreconf dnsmasq-base ssl-cert libssl-dev \
                      libxcb-present-dev libcairo2-dev libpango1.0-dev \
                      libxcb-render0-dev apache2-dev apache2-bin \
                      libev-dev apache2 libprotobuf-dev protobuf-compiler \
                      libboost-dev libboost-system-dev texlive ntp ntpdate
## 2. Install Python 2.7 and Setup pip
 #### Install Python 2.7
sudo apt-get install -y python2.7 python2.7-dev
 #### Get pip installer for Python 2.7
curl https://bootstrap.pypa.io/pip/2.7/get-pip.py -o get-pip.py
python2.7 get-pip.py --user

# Add pip to your PATH
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

#### Install required Python packages
pip2.7 install numpy matplotlib pyyaml tabulate pandas
## 3. Clone Pantheon Repository
 #### Clone Pantheon repository
git clone https://github.com/StanfordSNR/pantheon.git
cd pantheon

#### Initialize submodules
git submodule update --init --recursive
## 4. Install and Configure MahiMahi
 ### Clone MahiMahi repository
cd ~
git clone https://github.com/ravinet/mahimahi.git
cd mahimahi

#### Generate configure script
./autogen.sh

#### Configure build
./configure

#### Build the software
make

#### Install system-wide
sudo make install

#### Verify installation
mm-delay --help
## 5. Setup Pantheon
 Go to pantheon folder
cd ~/pantheon

#### Make scripts executable
chmod +x -R ./src/
chmod +x tools/install_deps.sh

#### Run Pantheon's dependency installer
./tools/install_deps.sh

#### Fix Python paths in Pantheon scripts (Python 2.7 modification)
find ./src -type f -name "*.py" -exec sed -i '1s|#!/usr/bin/env python|#!/usr/bin/env python2.7|' {} \;

#### Enable IP forwarding (required for network emulation)
sudo sysctl -w net.ipv4.ip_forward=1

#### Load necessary kernel module
sudo modprobe ifb

#### Setup congestion control schemes
python2.7 ./src/experiments/setup.py --setup --schemes "cubic bbr vegas"
python2.7 ./src/experiments/setup.py --schemes "cubic bbr vegas"
## 6. Create Custom Trace Files
#### Create directory for custom traces
mkdir -p ~/pantheon/custom_traces

# Create a 50 Mbps trace file
python2.7 -c "
for i in range(5000000):
    ms = int(i * 0.24)  # A packet every 0.24 ms (1500 * 8 / 50,000,000)
    print(ms)
" > ~/pantheon/custom_traces/50mbps.trace

# Create a 1 Mbps trace file
python2.7 -c "
for i in range(100000):
    ms = int(i * 12)  # A packet every 12 ms (1500 * 8 / 1,000,000)
    print(ms)
" > ~/pantheon/custom_traces/1mbps.trace
## Running Experiments
## 1. High-Bandwidth, Low-Latency Test
 #### Run high-bandwidth test (50 Mbps, 10 ms RTT)
python2.7 ./src/experiments/test.py local --schemes "cubic bbr vegas" \
  --data-dir high_bw_results \
  --prepend-mm-cmds "mm-delay 5" \
  --uplink-trace ~/pantheon/custom_traces/50mbps.trace \
  --downlink-trace ~/pantheon/custom_traces/50mbps.trace \
  --runtime 60
## 2. Low-Bandwidth, High-Latency Test
 #### Run low-bandwidth test (1 Mbps, 200 ms RTT)
python2.7 ./src/experiments/test.py local --schemes "cubic bbr vegas" \
  --data-dir low_bw_results \
  --prepend-mm-cmds "mm-delay 100" \
  --uplink-trace ~/pantheon/custom_traces/1mbps.trace \
  --downlink-trace ~/pantheon/custom_traces/1mbps.trace \
  --runtime 60
## 3. Analyze Results
 #### Analyze high-bandwidth results
python2.7 ./src/analysis/analyze.py --data-dir high_bw_results

#### Analyze low-bandwidth results
python2.7 ./src/analysis/analyze.py --data-dir low_bw_results


# File Modifications
During setup, the following modifications were made to work with Python 2.7:

### Changed all Python script headers from:

#!/usr/bin/env python
to:
#!/usr/bin/env python2.7

This modification was necessary because the Pantheon framework was designed to work with Python 2.7, but modern Ubuntu systems use Python 3 as the default Python interpreter.
