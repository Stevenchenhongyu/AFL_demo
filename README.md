# Fuzzing Web Application
Author:

### Table of contents
- [Introduction](#1-introduction)
    - [Overview](#Overview)
    - [Priority of Development](#Priority-of-Development)
- [Setup](#2-setup)
- [Current Progress](#3-current-progress)
- [Module Explanation](#4-module-explanation)
    - [Upload File](#upload-file)
    - [Start Fuzzing Job](#start-fuzzing-job)
    - [Status Monitoring](#status-monitoring)
- [Unfinished Modules](#5-unfinished-modules)
    - [Docker Healthcheck](#Docker-Healthcheck)
    - [Distributed System](#Distributed-System)
## 1) Introduction

---
### - Overview


This platform is implementated with fuzzing tools which are applied to help developers do fuzzing tests as a part of self-healing project. So far, this platform supports AFL's normal and LLVM fuzzing. Below is the UML to describe the function flow of the platform:
![UML](./README_images/UML.png)

### - Priority of Development

> 5=>Very Important & 1=>Not Important at all

| Status | Feature Description |
|------------------|------------------------|
| done | Accept Fuzz Jobs (Fuzz Binary + Meta Data + Test Cases) (5 for accepting Fuzz Jobs, 4, for Test Cases, 3 for Meta Data) |
|  |Schedule Fuzz Jobs according to their Priority (3) |
|done| Execute Fuzz Jobs (5)|
|  | Minimize Failing Test Cases of a Fuzz Job (2)|
|  | Minimize the passed Test Case set of a Job (2)|
| done | Maintain the results of an executed Job (Logs, Test Cases, Manifest) (5 for Logs, 4 for Test Cases, 3 for Manifest) |
|  | Support removal of scheduled, but not yet executed Jobs (1) |
|  | Support the interruption of the currently executed Job(s) (1) |
| done | Support to display the current status (running, pending Fuzz Jobs) (1) |
| done | Provide Results of executed Fuzz Job (e.g. all Results with certain Meta-Data) (5 for results, 2 for filtering according to Meta-Data) |
|   |Support the removal of obsolete Job Results (1) |
|   | Provide an alerting mechanism on a failed Job (1) |
## 2) Setup
---
All pre-required dependencies to start this web serivece are documented in the _requirement.txt_. But for Python and Django, you need to install by yourself. Docker is pre-installed in server already. But if you want to try this server out, you need to check it out by yourself.
1. prepare virtual enviroment 
```bash
python3 -m venv continental_afl_framework
```
2. activate virtual enviroment and install django
```bash
source continental_afl_framework/bin/activate
```
3. install all dependencies
```bash
pip install -r requirements.txt
```
4. migrate database 

It is recommended to  __stop all tasks__ after you finish your daily work. If not, output of fuzzing will be accumulated and fuzzing cannot auto overwrite it when it meets some limits. Therefore, you need to manually remove that output folder when you try to create tasks with old id after __"database flush"__ 
```bash
python manage.py makemigrations

python manage.py migrate

python manage.py flush
```
5. start the server

IP address and port are flexiable to change
```bash
python manage.py runserver 10.219.39.132:8040
```
## 3) Current Progress
---
Here is an introduction about how to use this platform to do a fuzzing testing.

> Home page of our platform and users could monitor their tasks here:
![home](./README_images/home.png)

> By clicking the "Create new job" button, users are directed to decision page. Users could choose which fuzzing mode to use. Due to we only have AFL so far, we did not add a fuzzer choosing page before this page.
![home](./README_images/decision.png)

> After choosing the fuzzing mode, users could see the upload form. Then choose the fuzzing target and all other needed files to start the fuzzer. For zip upload, you just need to upload one zip file and name the task but you need to make sure your manifest file could match with your materials
![home](./README_images/upload.png)

> Detail page of our platform. User could access this page by clicking the task title in home page. Coverage Calulation is not activated yet, under development. 
![home](./README_images/detail.png)

> Crashes list of fuzzer output. Users could remove the crash once they finish the debug and patch.
![home](./README_images/Crashes_list.png)

> Output of ASAN. Users could check error log here to help them debug. Users could access this page by clicking the crash seed title in crash_list page.
![home](./README_images/crash_detail.png)

## 4) Module Explanation
---

- [Upload File](#upload-file)
- [Start Fuzzing Job](#start-fuzzing-job)
- [Status Monitoring](#status-monitoring)

### - Upload File
---
- Upload by selecting file one by one

 Users must upload their [fuzz_target](https://afl-1.readthedocs.io/en/latest/instrumenting.html) and [ASAN_fuzz_target](https://afl-1.readthedocs.io/en/latest/notes_for_asan.html)


- Upload by zip file 

Users need to have _manifest.txt_ file in zip file. This is the rough structure of zip file:
```
    input.zip
    |
    |- fuzzgoat.exe
    |- dictionary.txt
    |- in/
       |-corpus1.txt
       |-corpus2.txt

    |- manifest.txt
        {
            "fuzz_target": "fuzzgoat.exe",
            "ASAN_fuzz_target": "fuzzgoat_ASAN.exe",
            "dictionary" : "dictionary.txt", (optional)
            "corpus"     : "in/" (optional)
        }
```
### - Start Fuzzing Job
---
This process will call docker by shell scripts. I will explain some important codes in these related scripts. 


a) Dockerfile
```Docker
# The whole AFL is built on ubuntu 16.04
FROM ubuntu:16.04

MAINTAINER Steven Chen Hongyu <chenhongyusteven@gmail.com>
ENV http_proxy "http://uib13471:Conti2020!@cias3basic.conti.de:8080"
ENV https_proxy "http://uib13471:Conti2020!@cias3basic.conti.de:8080"
# install git
RUN apt-get update
RUN apt-get -y install git
# install gcc
RUN apt -y install build-essential
RUN apt-get -y install manpages-dev

# git clone afl from github and run
RUN git clone https://github.com/google/AFL.git
RUN echo "you already download afl in your container"

# install afl
# RUN cd AFL
WORKDIR /AFL
RUN apt-get -y install make
RUN make
RUN echo "you succeffuly install afl"

# link in folder and fuzzing target into volume
RUN mkdir vol

# RUN export "AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1"
# CMD export "AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1";./afl-fuzz -i ./vol/in -o ./vol/out ./vol/fuzzgoat @@
# copy shell script in container
COPY ./afl-fuzz.sh /AFL
COPY ./afl-plot /AFL
RUN chmod 777 afl-fuzz.sh

ENV AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1
ENV AFL_SKIP_CPUFREQ=1
RUN sleep 5

# the default value for starting fuzzing is 1
CMD ["1"]
# we could pass in the num to differentiate fuzzers
ENTRYPOINT ["/AFL/afl-fuzz.sh"]
```
"Sleep 5" is used to delay the time of starting the fuzzer. There is one confused bug before: If the uploaded file was in big size, AFL docker cannot execute it becaused it is occupied by other progress. To solve this issue, we add in the "sleep" command to give docker more buffer time to build the volume.


To enhance the performance of the platform, we plan to add in more fuzzing tools which means we may need to create more Dockerfiles. At the same time, because AFL is not updated for a long time, we choose to directly git clone it online. However, due to other fuzzing tools are keeping updated, we may also need to do version control for these Dockerfiles to ensure the performance of these Dockerfiles.


b)  invoke_container.sh 
```bash
#! /bin/bash
fuzzing_job_corpus_path=$1
fuzzing_job_bin_path=$2
fuzzing_job_parallel_threads=$3
fuzzing_job_id=$4

mkdir ./media/$fuzzing_job_id/$fuzzing_job_corpus_path
mkdir ./media/$fuzzing_job_id/out
mkdir ./file_upload/static/file_upload/images/$fuzzing_job_id
chmod 777 ./media/$fuzzing_job_bin_path
chmod 777 ./media/$fuzzing_job_id/ASAN_file

for i in $(seq 1 $fuzzing_job_parallel_threads)
    do
        docker_name=$(id -u)_afl_docker_${fuzzing_job_id}_$i
        docker run -d --name $docker_name -u $(id -u) -v $(pwd)/media:/AFL/vol afl_docker:4.0 $fuzzing_job_corpus_path $fuzzing_job_bin_path $i $fuzzing_job_id
    done
```
We need chmod to make fuzz_target(fuzzing_job_bin_path) and ASAN_fuzz_target(ASAN_file) executable.

Although we create volume between media and container, outputs from AFL in container are own by root. Therefore, to avoid inputting root password each time, we have "-u" to pre set the owner and group for all outputs from container.

c) afl-fuzz.sh 
```bash
fuzzing_job_corpus_path=$1
fuzzing_job_bin_path=$2
fuzzing_job_parallel_threads=$3
fuzzing_job_id=$4

./afl-fuzz -i ./vol/$fuzzing_job_id/$fuzzing_job_corpus_path -o ./vol/$fuzzing_job_id/out -M fuzzer$fuzzing_job_parallel_threads ./vol/$fuzzing_job_bin_path @@
```
This shell script is copied into afl docker image when we build afl docker image. Go thorugh this manual to better understand this script: [AFL manual](https://afl-1.readthedocs.io/en/latest/fuzzing.html)

### - Status Monitoring
---

a) afl-cov 

This script will return the code coverage of AFL but it is still under development. So far, we redirect the output of afl-cov to a txt file and then extract useful information and return it to frontend. However, this seems not a smart way. We plan to to direct the output directly to database or at least that we need to update database to track the performance. By the way, to enhance this feature, we need to ask users to upload fuzztarget compiled by gcc with pre set flags "-fprofile-arcs -ftest-coverage". Please refer to this [source](https://github.com/mrash/afl-cov)

This is the sample code in _file_upload/views.py_ but commented. __afl-cov should be activated before afl-fuzz starts__.
```python
cmds = [
            "./afl-cov",
            "--background",
            "-d",
            "{}/media/{}/out/".format(curr_path, str(fuzz_job.id)),
            "--live",
            "--coverage-cmd",
            "{}/media/{}/{} AFL_FILE".format(curr_path, str(fuzz_job.id),fuzz_job.binary_cov),
            "--code-dir",
            "{}/media/{}/".format(curr_path, str(fuzz_job.id),
            "--overwrite",
            "--cov-output-id",
            "{}".format(str(fuzz_job.id)),
        ]
subprocess.call(cmds)
```

Problem Statement: to implement the function of codes coverage calculation before demo, we simply defined the way of how it work. Therefore, there are some bugs need to be fixed:

* We deceptively give afl-cov pre fixed PID to force it to run when it finds the status of afl-fuzz:
    * SOLUTION: check the existence of container by PID?
```bash
def is_afl_fuzz_running(cargs):

    pid = None
    stats_file = cargs.afl_fuzzing_dir + '/fuzzer_stats'
    #print("checking afl liveness")
    #print("stats_file location: "+stats_file)

    if os.path.exists(stats_file):
        pid = get_running_pid(stats_file, 'fuzzer_pid\s+\:\s+(\d+)')
        print("pid is {}".format(pid))
        # TODO: check docker
        # FIXME: dirty hack
        pid = 999
    else:
        for p in os.listdir(cargs.afl_fuzzing_dir):
            stats_file = "%s/%s/fuzzer_stats" % (cargs.afl_fuzzing_dir, p)
            #print("afl-stats file exists in fuzzer1")
            if os.path.exists(stats_file):
                ### allow a single running AFL instance in parallel mode
                ### to mean that AFL is running (and may be generating
                ### new code coverage)
                pid = get_running_pid(stats_file, 'fuzzer_pid\s+\:\s+(\d+)')
                pid = 999
                if pid:
                    break

    return pid
```

* We used subprocess call instead of Popen, which could result in collision between multi afl-cov process
    * SOLUTION: use container or Popen
* We redirected the output from afl-cov to txt file and then call API "details" to extract the info from these txt files. To save the power in file management for future, we need to redirect this output to database directly.
    * SOLUTION: we could try redis which is more stable to use when we have multi subprocess of afl-cov write output to database
```bash
def log_coverage(out_lines, log_file, cargs):
    for line in out_lines:
        m = re.search('^\s+(lines\.\..*\:\s.*)', line)
        if m and m.group(1):
            logr("    " + m.group(1), log_file, cargs)
            id_of_task = cargs.cov_output_id
            #os.mkdir("./media/{}/cal_cov".format(str(id_of_task)))
            cov_txt=open("./media/{}/cal_cov/cov_lines.txt".format(id_of_task),"r+")
            cov_txt.write(m.group(1))
            cov_txt.close()
        else:
            m = re.search('^\s+(functions\.\..*\:\s.*)', line)
            if m and m.group(1):
                logr("    " + m.group(1), log_file, cargs)
            else:
                if cargs.enable_branch_coverage:
                    m = re.search('^\s+(branches\.\..*\:\s.*)', line)
                    if m and m.group(1):
                        logr("    " + m.group(1),
                                log_file, cargs)
    return
```
b) How to get the crash dashboard
```python
def ASAN(request, FuzzingJob_id):
    """Reprsent all crashes in detailed way

    Args:
        request (http request): HTTP request sent by user
        FuzzingJob_id (int) : id of FuzzingJob to download

    Return:
        response(http response): HTTP response containing all crashes
    """

    fuzzing_job = get_object_or_404(FuzzingJob, pk=FuzzingJob_id)
    task_id = fuzzing_job.id
    task_name = fuzzing_job.description
    txtfiles = []
    for thread_num in range(1, fuzzing_job.parallel_threads + 1):
        for file in glob.glob(
            "{}/media/{}/out/fuzzer{}/crashes/id*".format(
                curr_path, task_id, thread_num
            )
        ):
            file = ntpath.basename(file)
            txtfiles.append(file)

    txtfiles = natsort.natsorted(txtfiles, reverse=False)
    return render(
        request,
        "ASAN.html",
        {"fuzzing_job": fuzzing_job, "files": txtfiles},
    )

```
This part will return a list containing all crashes.

```python
def ASAN_detail(request, FuzzingJob_id, crash_id):
    """Reprsent all crashes in detailed way

    Args:
        request (http request): HTTP request sent by user
        FuzzingJob_id (int) : id of FuzzingJob to download

    Return:
        response(http response): HTTP response containing all crashes
    """
    fuzzing_job = get_object_or_404(FuzzingJob, pk=FuzzingJob_id)
    task_id = fuzzing_job.id
    task_name = fuzzing_job.description
    for thread_num in range(1, fuzzing_job.parallel_threads + 1):
        if os.path.isfile(
            "{}/media/{}/out/fuzzer{}/crashes/{}".format(
                curr_path, task_id, thread_num, crash_id
            )
        ):
            with open(
                "{}/file_upload/static/file_upload/images/{}/ASAN.txt".format(
                    curr_path, task_id
                ),
                "w",
            ) as output_file:
                cmds = [
                    "{}/media/{}/ASAN_file".format(curr_path, task_id),
                    "{}/media/{}/out/fuzzer1/crashes/{}".format(
                        curr_path, task_id, crash_id
                    ),
                ]
                subprocess.call(cmds, stderr=output_file)

        else:
            continue

    return render(
        request,
        "ASAN_detail.html",
        {"fuzzing_job": fuzzing_job},
    )
```
We store the ASAN output in a txt file and let frontend directly read it.

## 5) Unfinished Modules
---
- [Docker Healthcheck](#Docker-Healthcheck)
- [Distributed System](#Distributed-System)

### - Docker Healthcheck

To better understand docker healthcheck, please go through this [doc](https://docs.docker.com/engine/reference/builder/)

Becuase we run fuzzer in containers, we need some ways to monitor the health of all container in case that container is shutdown accidently but status of it is not successfully updated. At the same time, we also need to enhance this feature to make sure "restart" function could work nicely. 

In /file_upload/models.py, we defined 6 status for our tasks:
```python
JOB_STATUSES = (
        ("W", "Waiting"),
        ("R", "Running"),
        ("C", "Completed"),
        ("E", "Error"),
        ("P", "Paused"),
    )
```
To improve the performance of this platform, we need to make our backend update these status based on health situation of our containers.

### - Distributed System

We planned to run our tasks over more than one machines which means we need container orchestration. We realize some ways but haven't tried it out, such as Ansible, Dockerswarm and Kubernetes.


























# Todos

1. When options for fuzzing other than Normal is setup, consider using templates and models to generate forms instead of writing out the entire form. 
    1. When there are more options we will have to create more forms. 
    2. If we use templating we will have cleaner code (and less code to write), since we will not need a path for every form. See [documentation](https://docs.djangoproject.com/en/3.1/intro/tutorial04/)
    3. We can only write it afterwards, instead of now, since we're not decided on all the other options to have -> we're unclear on which fields to provide for in the template etc.
2. Streaming output [stackoverflow link](https://stackoverflow.com/questions/63035915/django-subprocess-continuous-output-to-html-view)
    1. Option 1: StreamingHttpResponse
    2. Option 2: Django Channels (websockets)
3. Tie in database
    1. Generate model
    2. Modify function to create model and refer to model while working
4. Docker
    1. [Tutorial](https://docs.docker.com/compose/django/)
5. Tests
    1. Test that uploaded files exist
    2. Test that the script is called ?
    3. Test that the webpage is correct
    4. Test that form works in failing case?
6. View page
    1. Generate index page for jobs
7. Task queue
8. Initialisation
    1. When the project is initialised for the first time it may be missing some key files. Write a script to initialise these files.
9. Fuzz targets
    1. Find a proper fuzz target for use on demo on Monday
10. Test for integrity of job status
    1. Make sure that all elements in the FuzzingJob class have job status of either 'Running','Queued', or 'Completed'
11. 
    1. Pair pause/ start etc. job with shell script
12. 
    1. Break up God classes. 
    2. Review development plans and design job subclasses.
13. 
    1. Custom manager for jobs (to make jobs, run/pause/resume etc)
14. Validate manifest.txt in zipfiles in utils handle_zipfile
