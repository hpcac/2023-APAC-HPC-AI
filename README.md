# Frequently Asked Questions

## Q1. Is a specific project needed in my team's proposal?

Question: Whether or not a team is supposed to apply with a specific project. Or are we going to decide on the specific project after we had the training?  My team are not sure if our application should contain only the details of the team, without a specific project. 

> Answer: You can provide just the details of the team information, without a specific project. No need to register with your own specific project.

## Q2. What the training will be like?

Question: What the training will be like? Is it online lecture and will there be records of the lecture for ones who maybe available at training period?

> Answer: The trainings are live online lectures that will be recorded, and the links to the recordings will be shared for online replay.  Participants are encouraged to join the live meetings to interact with the instructors..



## Q3. Is it allowed to use reduced precisions in AI task?

Question: You recommended to use DeepSpeed for the AI task, but you also mentioned in the [task description](https://github.com/hpcac/2023-APAC-HPC-AI/blob/main/BLOOM-176B_Inference/BLOOM_Inference_Task_Description.md) that HF framework is also allowed. Just want to confirm that HF framework and int8 precision is allowed for the competition?



> Answer: Yes, the HF framework and int8 precision Bloom 176B are allowed for the competition.
>
> However, to reduce the amount of computing and skew the results is not allowed in this inference competition, since it would make the comparison of throughputs results between different competition teams meaningless.



## Q4. Is it allowed to reduce amount of computing in AI task?

Question: My optimized inference needs less computing but produce the same output. Is it still not allowed since it reduces the computation amount? 

> Answer: Your optimization is allowed, because as you said it produces the same output as the original inference..
>
> Actually such type of optimization is encouraged in the competition and will benefit the Bloom community.
>
> However, it is important for you to prove that the output for any input is identical to the original inference without any modifications; otherwise, the results will be deemed invalid.
