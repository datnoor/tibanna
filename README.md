# Tibanna
Tibanna is the gas mine in Cloud City that makes Hyperdrives zoom.  It's also the pipeline running in the cloud that ensure data is properly processed for 4dn.


# Goal
The goal is to construct a version zero framework for executing a cwl pipeline directly on aws without using SBG (nor DNA nexus), that can be used as an independent module.
 
# Summary diagram
The module works as a diagram below:
First, create a JOBID, a run.json file, a tiny shell script (run_workflow.sh) file and upload the run.json file to S3. (This part has yet to be implemented, currently I manually created the JOBID, run.json file and the shell script.)
Then, launch a self-executing EC2 instance with a JOB ID and run_workflow.sh passed to it.
The self-executing EC2 instance will download necessary scripts and workflow codes from github and run.json, data from S3 and run the workflow and upload the output and status to S3 and then terminate itself.
The log files will be sent to S3 intermittently but not in real time.

--diagram--

# Launching a self-executing EC2 instance
From anywhere you have aws configured with the right credentials and region, you can run the following command to launch an instance of the desired type. (For more details, see Test12. The content of ths page is an upgraded version of Test12.) 
Basically, this command launches an instance based on a CWL-Docker-toil AMI (AMI with docker daemon and cwl-runner and toil installed based on Amazon Linux AMI), with shut-down-behavior 'terminate' and read/write access to S3 and has run_workflow.sh as user-data. Those are critical requirements. Most likely an additional EBS volume must be attached (in the below example to 100GB, io1 type with 5000 IOPS) because the default 8GB is not sufficient for most data. The exact volume size can be determined based on the data size and the workflow (e.g. which determines intermediate and output file sizes). The instance type is set to i2.xlarge in the case below, but it could also be flexible depending on the data size.

```
JOBID=v989328isyrbag02
INSTANCE_TYPE=i2.xlarge
EBS_SIZE=100  ## in GB
EBS_TYPE=io1
EBS_IOPS=5000
AMI_ID=ami-7ff26968
KAYPAIR=duplexa.4dn
./create_run_workflow.sh $JOBID ## This creates the userdata script run_workflow.$JOBID.sh.

# make sure your json file is already on s3 before doing this:
aws ec2 run-instances --image-id $AMI_ID --instance-type $INSTANCE_TYPE --instance-initiated-shutdown-behavior terminate --count 1 --monitoring Enabled=true --enable-api-termination --block-device-mappings DeviceName=/dev/sdb,Ebs="{VolumeSize=$EBS_SIZE,VolumeType=$EBS_TYPE,Iops=$EBS_IOPS,DeleteOnTermination=true}" --iam-instance-profile Arn=arn:aws:iam::643366669028:instance-profile/S3_access --ebs-optimized --user-data file://run_workflow.$JOBID.sh --key-name $KEYPAIR >> launch.$JOBID.log
```
 
The same kind of command can be executed to launch an instance in other ways (e.g. using python, with different security handling, etc, but the requirements stated above must be kept.)
Once you call the EC2 instance, the rest is completely independent of how you called it.

For example, ``run_workflow.v989328isyrbag02.sh`` looks as below:
```
#!/bin/bash
JOBID=v989328isyrbag02  
RUN_SCRIPT=aws_run_workflow.sh
SCRIPT_URL=https://raw.githubusercontent.com/hms-dbmi/tibanna/master/
wget SCRIPT_URL/$RUN_SCRIPT
chmod +x $RUN_SCRIPT
source $RUN_SCRIPT $JOBID
```
The second line should depend on the JOBID and this script should be generated on the fly by create_run_workflow.sh, after a JOBID is assigned. This script will be passed to EC2 and executed at the beginning. It will first download aws_run_workflow.sh from github and run it with the specified JOBID. The rest will be taken care of by aws_run_workflow.sh.
 
# Scripts that will be downloaded to the worker instance
Basically, aws_run_workflow.sh downloads two python scripts that parses and updates json files from github and these three scripts together will do all the works and terminate the EC2 instance once everything is finished.
The three codes are:

 ``aws_run_workflow.sh``

 ``aws_decode_run_json.py``

 ``aws_update_run_json.py``

 
# Assumptions (requirements)
The only assumptions required for this module to work are as below:
1) Must have the above three scripts in the specified SCRIPT_URL. This url can change, if it is changed inside both create_run_workflow.sh and aws_run_workflow.sh. (hard-coded once in each script as an environmental variable)
2) The run json file must be in S3 bucket named 4dn-aws-pipeline-run-json. This bucket name can change, if it is changed inside aws_run_workflow.sh (appearing once in the script as an environmental variable).
3) The run json file name must be $JOBID.run.json (replace $JOBID with the actual job ID).
4) The run json file must be in the following format:
```
{
 "Job": {
   "JOBID": "v989328isyrbag02", 
   "App": {
       "App_name": "Gitar",
       "version": "0.2",
       "cwl_url": "https://raw.githubusercontent.com/SooLee/gitar.workflow/master/cwl.draft3_v0.2/",
       "main_cwl": "hictool-bam2hdf5.cwl",
       "cwl_files": []
   },
   "Input": {
       "Input_files_data": {
           "input_bam1" : {
               "class": "File",
               "dir": "4dn-tool-evaluation-files",
               "path": "GM12878_SRR1658581_pair1.bam"
           },
           "input_bam2" : {
               "class": "File",
               "dir": "4dn-tool-evaluation-files",
               "path": "GM12878_SRR1658581_pair2.bam"
           }
       },
       "Input_files_reference": {
           "bowtie_index" : {
               "class": "File",
               "dir": "4dn-tool-evaluation-files",
               "path": "hg19.bowtieIndex.tgz"
           },
           "chrlen_file" : {
               "class": "File",
               "dir": "4dn-tool-evaluation-files",
               "path": "hg19.chrlen_file"
           },
           "RE_bed" : {
               "class": "File",
               "dir": "4dn-tool-evaluation-files",
               "path": "HindIII_hg19_liftover.bed"
           }
       },
       "Input_parameters": {
           "contact_matrix_binsize": 50000,
           "chromosome": ["1","2","3","4","5","6","7","8","9","10","11","12","13","14","15","16","17","18","19","20","21","22","X"]
       }
   },
   "Output" : {
       "output_bucket_directory": "4dn-tool-evaluation-files/output/20160711.v989328isyrbag02"
   },
   "Instance_type": "i2.xlarge",
   "EBS_SIZE": "800",
   "EBS_TYPE": "io1",
   "EBS_IOPS": 5000,
   "AMI_ID": "ami-78c13615",
   "start_time" : "20160711-155396"
 }
}
```

The fields required by the script include
* "App" / "cwl_url" : name of github directory that contains the cwl files (directory for raw files).
* "App" / "main_cwl" : name of the cwl file to run (wither workflow cwl or a command (single-step) cwl).
* "App" / "cwl_files" : name of the other cwl files needed to run (e.g. called by main_cwl).
* "Output" / "output_bucket_directory" : name of bucket and subdirectories where output files will be put. The bucket must exist. Subdirectories can be created on the fly.
* "Input" / "Input_files_data" / "class" : always 'File' unless array of files. It follows CWL convention used to define a file input in the meta data file used by cwl-runner.
* "Input" / "Input_files_reference" / "class"  : same.
* "Input" / "Input_files_data" / "dir" : name of bucket and subdirectories where a corresponding input data file is stored.
* "Input" / "Input_files_data" / "path" : name of the input data file. It's called 'path' following the CWL convention but it actually means just the file name (without directories).
* "Input" / "Input_files_reference" : Same structure as "Input_files_data". "Input_files_data" and "Input_files_reference" are not differentiated in these scripts. This distinction is mainly for (potential) convenience when generating this json file, because Input_files_data should contain actual data files like fastq and bam files, whereas Input_files_reference should contain non-experimental data like bowtie index, restriction enzyme site file, chromosome length file, etc, which can be chosen by the user or could be dependent on the experiment (e.g. species, restriction enzyme used, etc) but will constantly reside in some S3 bucket and shared across runs. If there is no reference file, still include this field with null as its value.
* "Input_parameters" : In case the workflow has input parameters other than files, then they can be entered. For a parameter that is scattered according to the workflow, you can use an array representation in square brackets (e.g. chromosomes in the above example)
Other fields are good to have, but not assumed by the above three scripts.
 
# Output
It produces and uploads the following output files in the specified output directory in S3, in addition to the actual output files from the pipeline execution.
* log : stdout and stderr capture from all commands run on the EC2, up to the point before output files are uploaded to S3.
* md5sum.txt : md5sum of all output files except log and md5sum.txt itself.
* JOBID.postrun.json : This is the same as run.json but with added information of end time and status. The status is either zero (if every command run on EC2 had exit status zero) or a string of exit statuses (e.g. 21,0,0,0,0,1,0,1,1,1,1) that represent the exit status of the commands in the order of execution.
 
