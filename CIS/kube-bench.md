# kube-bench Tool

Kube-bench tool is an open source tool from Aqua Security that can perform automated assessments to check wheather kubernetes is deployed as per Security Best Practices. 

So there are different way to get staeted with kube-bench
1. Deploy as a Docker Container
2. Deploy as a pod in a K8s cluster like jobs
3. install the kube-bench binary
4. compile from source-code 


labs:
* go to kube-bench github page
* Identify latest release binary
* Install kube-bench on Control-Plane Node
* Run Assessment and review the result
* Fix Issues 

---------------------------
Q. Install version `0.12.0` of kube-bench. Refer to the documentation link bookmarked above the terminal under the name `kube-bench Release`.

Ans. Download the kube-bench_0.12.0_linux_amd64.deb binary: `https://github.com/aquasecurity/kube-bench/releases/tag/v0.12.0`

`curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.12.0/kube-bench_0.12.0_linux_amd64.deb -o kube-bench_0.12.0_linux_amd64.deb`

Then install it using dpkg: `sudo dpkg -i kube-bench_0.12.0_linux_amd64.deb`

To verify, run the following and you should expect to see 0.12.0 as an output: `kube-bench version`

-------------------------

Run a kube-bench test for all targets and observe the results. `kube-bench run`

`kube-bench run --targets="master,etcd"`

`kube-bench run --targets="master"`