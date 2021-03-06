
----

# Project 1: Object recognition `S3, Lambda Layers, Rekognition, OpenCV` 

You'll create a computer vision workflow that intelligently detects dogs, kids (and arbitrary objects) in long streams of video.

## Introduction

### Motivation

You have a motion-sensing camera installed at your porch. When a motion is detected, the camera starts filming and stores it as separate video file to S3.
At the end of the day you want to know when the dog or kid roamed around too far this day. You can't be bothered to look through the videos manually. 
To complicate things, the camera often activates on birds, neighbors, delivery people.

As most IoT devices, the camera does not do any preprocessing. Therefore the video files are very large. Applying computer vision to the entire videos is infeasible.

Your workflow should efficiently detect at which times of the day your kid or dog appeared. It should be scalable (with number of videos) and cost-efficient (do with as few expensive tasks - Image recognition - as possible).

### Input 

A folder of videos of that day on S3.

```
└── your-video-bucket
    ├── 1604670378.mp4
    └── 1604670406.mp4
```


### Rough steps 

* Extract frames (every 0.5 second) from the videos and store them as images to S3. 
* Between each two subsequent images, check if there is a significant delta (something happening). 
* Use AWS Rekognition to detect dogs or kids on those
* Output when either appeared on the porch that day as human sentence (string).



## Week A (Homework 06): Sketch the workflow with AFCL


Sketch a preliminary workflow with AFCL. 

Think about these things:

* How to group the above algorithm into functions 
* What you can do in parallel; what is independent from each other
* What information each function needs, how you best represent it (S3 ARNs, collections thereof, named fields)
* How information flows between your functions


Put together the workflow using the AFCL Java API. 
It's preliminary so you can (and will probably) make changes next week. The main goal is to think about modularity and data flow. Since different team members will develop the functions, it's helpful to think about interfaces between them early on.

Create empty functions that just produce the data how you specified with AFCL. Run the workflow with these functions with the Enactment Engine.


## Week B (Homework 07): Code the functions

### Rough functions

#### Frame extractor `S3, Lambda Layers` `Python 3.8`

This function should load a `.mp4` from S3, extract frames (every 0.5 second) as image using OpenCV and store the images to S3. 

Hints:
1. Make yourself familiar with OpenCV for Python and try it out on your Laptop. 
1. To avoid uploading OpenCV for each function, and everytime you change them, explore other ways to get the library into your functions. 
1. Stick to a deterministic naming convention for everything your store to S3, to avoid overwrites.


#### Delta finder `S3, Lambda Layers` `Python 3.8`

This function should take images from S3 and find interesting frames by doing delta detection with OpenCV on each two subsequent frames.
Return frames where the delta is above a threshold.

Hints:
* Find a way to know what images are 'subsequent'.
* You can get more reliable deltas by [smoothing](https://towardsdatascience.com/types-of-convolution-kernels-simplified-f040cb307c37) the images with [`filter2D`](https://pythonexamples.org/python-opencv-image-filter-convolution-cv2-filter2d/) before you compare them. 


#### Recognition and Interpretation `S3, Rekognition, SDK`

This function should use AWS Rekognition to find dogs or kids in frames that Delta finder deemed interesting.
Make sure you also retain the information whether it was a dog or kid.

Hints:
* Choose an appropriate confidence threshold



#### Grouping and Output

This function should construct a human sentence that tells you when the dog or kid appeared in the shot that day. It should be [brief and relevant](https://developer.amazon.com/en-US/alexa/branding/alexa-guidelines/communication-guidelines/brand-voice). For that purpose, you will have to intelligently group by time (`'at 2pm'`), truncate if necessary (`'5 times on the afternoon'`), and so on.
Return the sentence as string.


## Week C (Homework 08)

Orchestrate the functions with the Enactment Engine.


----



# Project 2: BWA `S3, bwa & samtools, Integrative Genomics Viewer` 


You'll create a real-life workflow to process Escherichia Coli DNA samples, and then investigate whether the patient can be treated with antibiotics.

## Introduction

#### Motivation

You are working in a lab that frequently receives DNA samples of Ecoli bacteria from a hospital.
Ecoli can cause food poisoning, and the recommended treatment for healthy adults is to just wait it out.
However, in some cases antibiotics may be necessary.

You want to find out if a Ecoli sample is resistant to antibiotics, to suggest a treatment for the patient.
This has to happen as fast as possible from when the sample arrives. 

Ecoli DNA is fairly short (4.6 million base pairs). However, samples are usually contaminated (human, bacterial DNA) so processing is hard and has to be parallelized.

Your lab uses a short-read sequencer such as [Illumina MiSeq](https://www.illumina.com/systems/sequencing-platforms/miseq.html) to read ('sequence') the [basepairs](https://en.wikipedia.org/wiki/Base_pair#Examples) of the DNA. 


#### Input


```
└── your-bucket
    ├── NC_000913.3.fasta  
    └── reads
        ├── hipa7_reads_R1.fastq
        └── hipa7_reads_R2.fastq
```

* A [FASTA](https://genome.sph.umich.edu/wiki/FASTA) text file containing the entire DNA of Ecoli ('reference genome')
* A FASTQ text file (FASTA plus likelihood that reads are correct) with paired-end reads ('ABCDE' and 'EDCBA') of your Ecoli sample, obtained from the MiSeq.



#### Rough steps


* Split the reference genome into smaller parts
* Run bwa index  (parallel for each part)
* Run bwa aln (parallel for each part)
* Run bwa sampe (parallel for each part)
* Run samtools merge to concat .sam files into one .sam
* Run samtools sort to sort entries
* Run samtools view to convert to .bam (binary .sam)
* Run samtools index to make it searchable

#### Crash course in Bioinformatics

DNA is the source code of organisms; it's a long string of the bases A, T, G, C (compared to 0 and 1) that describes all traits of the organism. It's incredibly long - 2 meters and 3 billion basepairs (bits) - but it is bundled up ingeniously so it fits into cells that are just 10 to 100 micrometers in diameter.
Every cell contains an entire copy of the DNA.
To see if a human has brown eyes, a bacterium is resistant to antibiotics, or a crab may develop porous shields, we can just read its DNA - given we know how to interpret it.
Of course, every organism, including twins, has slightly different DNA.

DNA readers give you many random substrings of the DNA sample, whether that is hair, skin cells, or a bacterial colony.

We usually also have a reference DNA of that organism. This is akin to the picture on a puzzle box, and gives us a rough idea how the puzzle pieces (reads) fit together.

. | notes
---|----
 `ATCGAAACTT` | reference DNA
  `??????????` | skin sample DNA
 `ATCC`, `ACT`, `AAC`, `CCAAA`, `ACTT` | reads of skin sample
 <code>ATCGAAACTT</code> (reference DNA)<br><code>ATC<b>C</b>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><br><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ACT&nbsp;</code><br><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AAC&nbsp;&nbsp;</code><br><code>&nbsp;&nbsp;C<b>C</b>AAA&nbsp;&nbsp;&nbsp;</code><br><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ACTT</code><br><code>ATC<b>C</b>AAACTT</code> (inferred skin sample DNA) | what bwa does
We know that a `C` mutation at the fourth<br> position leads to brown eyes. Now we <br>check for that by hand in the inferred DNA. | IGV (by hand)


#### Hints

This workflow is all about file management at scale. Assume that every step (`bwa index`, `bwa aln` and so on) produces new files that the next step needs.
Furthermore, the parallel section produces files with the same name. We left you a recommendation how to handle this [in Week B](#rough-functions-1).



## Week A (Homework 06): Sketch the workflow with AFCL


Sketch a preliminary workflow with AFCL. 

It helps to try out [`bwa`](http://manpages.ubuntu.com/manpages/bionic/man1/bwa.1.html) on your Laptop. 

Think about these things:

* How to group the above algorithm into functions 
* How do you best 'keep' / 'transport' files between subsequent functions?
* Each parallel function will produce files with the same name. How do you prevent naming conflicts on S3?
* What information each function needs, how you best represent it (file content strings, S3 ARNs, collections thereof, named fields)
* How information flows between your functions


Put together the workflow using the AFCL Java API. 
It's preliminary so you can (and will probably) make changes next week. 

Create empty functions that just produce the data how you specified with AFCL. Run the workflow with these functions with the Enactment Engine.



## Week B (Homework 07): Code the functions. 

Make sure you can run the steps on your Laptop / PC.


### Rough functions

Serverless functions always see a fresh filesystem, but for `bwa` it's useful to have files persist.
We recommend to write a simple abstraction that stashes & fetches the `tmp` folder to an S3 folder, and thus fakes continuity between functions (at least within the `ParallelFor`).

#### Split `S3, Lambda Layers` `Python`

This function should split the reference genome `NC_000913.3.fasta` into smaller parts. You can use any library, binary, or shell command you find online to split FASTQ or FASTA files.

#### bwa index `S3`

This function should run `bwa index` on a reference genome split created previously. This creates an index to make it better searchable.

#### bwa aln `S3`

Run `bwa aln` (align) for for a reference genome split created previously, for both read files (5'3 and 3'5, here R1 and R2). This is the main step of `bwa`; it aligns the reads to a reference genome. It is visualized in the table above.


Hints:
* 5'3 and 3'5 are directions on DNA strings, similar to 'upstream' and 'downstream' of a river. In paired-end sequencing, you read DNA from both directions (hence `hipa7_reads_R1.fastq` & `hipa7_reads_R2.fastq`). Thanks to the [structure of DNA](https://en.wikipedia.org/wiki/Directionality_(molecular_biology)), you always know what 5'3 and 3'5 is, regardless of how you look at it.


#### bwa sampe `S3`

This function should run `bwa sampe` (**sam**-**p**aired-**e**nd) for the `.sai` pair created in the previous step (R1 and R2). This will put the aligned reads into one `.sam` file.

#### samtools merge

This function should run `samtools merge` to concat the `.sam` file of each reference genome split.

#### samtools sort, view

This function should run `samtools sort` to sort the `.sam` file.
Then, run `samtools view` to convert to a binary representation (`.bam` file).


## Week C (Homework 08)

Investigate the results - use the Integrative Genomics Viewer to analyze the genes for antibiotic resistance.

Orchestrate the functions with the Enactment Engine to run the functions automatically. Convert the AFCL to CFCL.

----

# Project 3: Prediction of stock prices `S3, Forecast, webhook`

You'll create a real-life workflow to predict the price of stocks you can buy and sell.

## Introduction

### Motivation
You frequently trade with a wide variety of stocks on various stock exchanges. Every morning, your assistant compiles you a list of 50 stock that have performed interestingly and might be worth a look.

You want to see at a glance how these stocks could perform.
Therefore you program a workflow that takes these names, predicts their prices, and visualises their past and projected price.

This should be done in parallel, per stock. The result should be visualised together.

Furthermore, functions should learn from their RAM usage, so that the workflow becomes more cost-efficient everytime you run it.
This becomes relevant in real-life cases where workflows are run millions of times.


### Input

```
['AAL', 'ALT', 'NCMI', ... ]
```

* A list of stock ticker symbols traded on some exchange.


### Rough steps

* Pull commodity prices to S3
* Enter them into AWS Forecast
* Forecast for the coming year for each commodity
* Download the results
* Notify yourself that the prediction is ready

----

# Prooject 4: Multi-Objective Optimization `Opt4J` (Java)

You will deploy a parallalized optimization algorithm which optimizes a given problem with respect to multiple objectives.

## Introduction

### Motivation

In this project, you will be working with [Opt4J](http://opt4j.sourceforge.net/), an open-source framework developed for optimization research. Opt4J offers a modular toolbox of diverse optimization components which can be used to implement optimizations with different optimizers, problem representations, evaluation approaches, etc.

In this task, your goal is to (a) study how the optimization of a multi-objective problem via an evolutionary algorithm is implemented within Opt4J, (b) develop a workflow which makes it possible to deploy the evolutionary algorithm as a distributed application, and \(c) evaluate the speedup compared to an execution on your local machine.

### Input

- Optimizer configuration
- Problem configuration

(A Java project with an optimization problem implemented within Opt4J will be provided as a starting point for this task).

### Rough steps (Irrelevant for this task)

- Familiarization with the implementation of an evolutionary algorithm within the Opt4J framework. 
- Definition of a parallelization concept: 
	- Dividing the optimization into individual tasks
	- Definition the FC for the evolutionary optimization
- Implementation
	- Implementation of the FC
	- Implementation of the functions

----

# Project 5: Gate Change Alert `S3, Lambda Rekognition, DB` (Java, Python, Node.js)

You need to create a workflow that performs a series of actions after a gate of a specific flight has changed at an  airport. 

## Introduction

### Motivation

The workflow reads the information about the flight, the new gate and then loads all available passenger data (from a Database) of that flight. Also, checks the queue length of the security check. Thereafter, for every passenger from that flight that is already at the airport, the workflow reads the gps location and calculates the time to gate.

### Input 

An image of security check on S3.


### Rough steps 

- create a database in format you like (in a container, run it in a VM, use some DB service of AWS)
- create a map of the Innsbruck Airport with GPS locations
- distinguish the security and public area
- read the image from S3 and use it as an input for AWS Rekognition

----

# Parts you can use in addition

## Amazon

```
S3 Glacier
AWS Backup
DynamoDB (NoSQL)
ElastiCache (distributed in-memory DB)
QLDB (immutable transaction log)
Timestream (IoT)
Cloud Map (AWS resource query)
Chatbot (subscribed to SNS)
Polly  (Text-to-speech)
MediaConvert (from * to S3)
Kinesis video streams (from web, sdk, to S3, \*)
Lex (barebones Alexa)
Forecast (time-series)
Rekognition (Object and scene detection, facial, face comparison, text in image)
Comprehend (Natural text processing: entities, key phrases, personal data, language, sentiment)
Translate
Athena (big data query service, basically SQL)
Data exchange (miscellaneous data)
Sumerian (three.js in Browser)
SQS (pub-sub queue)
Greengrass (IoT, local / in-cloud processing)
```

## IBM Lite

```
Watson assistant (chatbot / text to some action)
Visual recognition (scenes, objects...)
Cloudant (JSON database)
Annotator for clinical data
Translate
Object storage
Natural language understanding
Speech to text (streaming transcription)
Text to speech
Tone analyzer (text sentiment)	
```
