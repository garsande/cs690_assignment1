Name - Sandeep Garg
Repo link - https://github.com/garsande/cs690_assignment1
SHA of commit - 65c4ff0313324db4c9cb776f044e0df46f5007fa

# CS 690 Assignment 1 Report: Replicating a Controlled Evaluation

## Part 1. Verification evidence

Command:

```text
python -m harness.verify
```

Paste the five `OK` lines here. Keep `results/verification.json` in your repository.

OK: loaded 20 frozen tasks
OK: dataset sha256 5d84176547cb679f4145676d1f4dfd5061bf3b9600904911da8e5700e82eee3b
OK: generated Python executed in Docker sandbox
OK: candidate network probe was blocked
OK: model/configuration metadata written to results/verification.json

## Part 2. Tests and code questions

Paste the final summary line of `pytest -q` here.

20 passed in 3.23s

Answer each question in your own words, in about 75 to 150 words. Base every answer on the code in this repository, and name the files and functions you describe.

### Q1. The path of one attempt

The 20 tasks live in tasks/cs690_eval20.json. Each task has four fields:

    id           a label such as "A1-001"
    entry_point  the name of the function the model must write
    prompt       the plain-English description sent to the model
    tests        Python assert statements that decide pass or fail

First step is for tasks.py class, using load_tasks() method, to parses each task into an internal Python object containing,id, entry_point, prompt,tests, metadata. This produces a list of task objects that the evaluation runner will iterate over.

Second step is for runner.py class using _save_prompt() method to atke the task object and constructs the exact prompt text that will be sent to the model. This is written to disk at prompts/a1-controlled-eval-fall2026/A/A1-001.txt

Third step is for runner.py class using run()  to send the constructed prompt to the model (e.g., gpt-5.6-luna) and save the raw model output to, results/experiment/candidates/A/A1-001/sample_1.raw.txt and save the code at results/experiment/candidates/A/A1-001/sample_1.py. This file is what will be executed in the sandbox.

Then runner.py creates one provider.py object per model condition and calls its generate method once per attempt. This is like a sampler which sends one request to a model API and records what came back.

Next runner.py uses the grader.py to do two small jobs. First extract_python() takes the Python code out of the model's answer text. and then grade_candidate() runs that code against the task's own tests, inside the Docker sandbox (harness/sandbox.py). The verdict is pass or fail with no partial credit.  run_source() starts a fresh container for each candidate code, passes the code and the tests in, and reads one verdict back out. The container is thrown away afterward.
Inside the docker container the program docker_entry.py does the actual running. The Dockerfile in describes the container image. ensure_image() builds that image automatically the first time it is needed.  The Dockerfile copies this one file into the sandbox image, and sandbox.py starts a fresh container for every candidate answer.
sandbox.py file returns a structured result object after running it inside a docker container.

Last step is for the runner.py, run() method to write a JSON object per line into results/experiment/raw_results.jsonl

The Docker sandbox is used because it guarantees security, isolation, reproducibility, and fairness. Running arbitrary model‑generated code directly would be unsafe to execute on a actual computer. Running it directly on your computer would give the model your files, your network, and your accounts.

### Q2. What is sent and what comes back


What is sent, for every request:

requested_model — Specifies which model the harness asks the provider to use.
prompt — The full constructed text sent to the model (instructions + entry point + tests).
temperature — Controls randomness: higher = more diverse outputs, lower = more deterministic.
top_p — Controls nucleus sampling: restricts token choices to the most probable subset.
max_output_tokens — Caps how many tokens the model is allowed to generate.
effort — Indicates how much reasoning the model is expected to apply (e.g., “none”, “medium”, “high”).
timeout_seconds — Limits how long the model may take before the request is aborted.
provider — Identifies which backend is being used (e.g., OpenAI, Azure, Anthropic).
seed — Controls reproducibility of sampling (when supported).


What comes back:

returned_model — The actual model that produced the output.
requested_model - the model name the harness asked for
text — The full text the model generated.
provider - which API produced it
input_tokens  -  prompt size in tokens, if reported
output_tokens — Number of tokens generated.
total_tokens   - the two added together, if reported
stop_reason — Why generation ended (completed, length, stop sequence, etc.).


### Q3. Same prompt, different answers

Temperature 1.0 introduces controlled randomness. At 1.0, the model samples from a wide range of plausible next tokens rather than always choosing the highest‑probability one. Even with the same prompt, the model may choose different valid next tokens, producing different implementations.
Each attempt is an independent sampling run. The harness does not reuse hidden model state between attempts.
Each attempt is a fresh call and thus three attempts can produce different code structures, different variable names, different failure/success outcomes.

Why this variation is intended:

This experiment is designed to measure how much a model changes its answer under identical conditions. How often a model succeeds across multiple tries and whether the model reliably produces correct code.Also, how temperature interacts with coding tasks. If the model always produced identical outputs, we would learn nothing about its reliability or variability. Three attempts at temperature 1.0 gives a meaningful sample of the model’s behavior.


These are the artifacts someone else would need to rerun your experiment and check your work.

1. Prompt file - prompts/a1-controlled-eval-fall2026/A/A1-001.txt
2. Raw model output -results/experiment/candidates/A/A1-001/sample_1.raw.txt
3. Extracted candidate code  - results/experiment/candidates/A/A1-001/sample_1.py
4. Raw results log (master record)  -  results/experiment/raw_results.jsonl

These fields allow anyone to reconstruct the exact run:


requested_model — what was asked for
returned_model — what actually answered
temperature
top_p
max_output_tokens
effort
input_tokens
output_tokens
stop_reason
provider
task_id
prompt_path
prompt_sha256
condition_id
experiment_id
candidate_path
raw_response_path
sample_index
sandbox_image
sandbox_network
With these fields, someone else can, reconstruct the exact prompt and rerun the same model with the same settings. Re‑execute the candidate code in the same sandbox image and confirm reproducibility


### Q4. pass@k by hand

Show your work for pass@1 and pass@2 with n = 3 and c = 1, the values `pass_at_k` returned, and the shortcut `1 - (1 - c/n) ** k` for k = 2.

1. For pass@1
n=3,c=1,k=1
using the combination formula

1- COMB(n-c,k)/COMB(n,k)
1- COMB(3-1,1)/COMB(3,1) = 1- COMB(2,1)/COMB(3,1)
= 1-   2/3 = 1/3

With 3 attempts and only 1 correct, if we randomly pick 1 attempt, we succeed with probability  1/3.
pass@1 = 1/3

2. For pass@2
n=3,c=1,k=2
using the combination formula

1- COMB(n-c,k)/COMB(n,k)
1- COMB(3-1,2)/COMB(3,2) = 1- COMB(2,2)/COMB(3,2)

= 1-   1/3 = 2/3

With 3 attempts and only 1 correct, if we randomly pick 2 attempts, we succeed with probability  2/3.
pass@2 = 2/3

3. Now calculate pass@2 using shortcut
n=3,c=1,k=2
1 - (1 - c/n) ** k
1 -(1-1/3) ** 2
1- (2/3) * (2/3)
1- (4/9) = 5/9

pass@2 = 5/9 using shortcut

Why the two pass@2 differs?
The shortcut assumes that you are making k independent draws with replacement with success probablity c/n. Each draw is like, sample one attempt, check if it’s correct, then put it back and try again. The true pass@k definition assumes that you choose k distinct attempts without replacement from the n actual attempts the model produced. You are sampling from a fixed multiset of c correct and n − c incorrect attempts.    
pass@k is defined over a finite set of n attempts sampled without replacement, while the shortcut formula assumes k independent drwas with replacement.


### Q5. Why whole problems are redrawn


Return a percentile bootstrap CI for macro pass@k.

    The resampling unit is a task. Each replicate draws len(task_counts)
    tasks with replacement. Individual attempts are never pooled or drawn
    on their own. The repetitions and seed arguments are always used as
    given, so the same results always produce the same interval.

    This is the procedure on the Week 2 slide "Where that range comes from":

    1. Build a pretend suite of the same size by drawing whole tasks from
       the real results at random, allowing repeats.
    2. Compute the suite score of that pretend suite and keep it.
    3. Do this `repetitions` times.
    4. Sort the kept scores, cut off the lowest and the highest
       (1 - confidence) / 2 share, and return the two cut points as
       (low, high). With confidence 0.95 that is 2.5 percent at each end.

## Part 3. Replication

Part 3 has no written section. Its evidence is the committed `results/experiment/` and `prompts/` folders, and the dollars you spent, which go in the Part 4 table.

## Part 4. Results

Take every number from `results/experiment/summary_A.json` and `results/experiment/summary_B.json`, not from the console. Dollars spent come from the Usage page of your OpenAI account. If your account does not show them, write `not available`. If it shows only one total for the whole run, write the total in row A and `included in A` in row B.

| Condition | Requested model | Returned model version | Attempts per task | Total attempts | pass@1 | 95 percent CI for pass@1 | pass@2 | Input tokens | Output tokens | Dollars spent |
| --- | --- | --- | ---: | ---: | ---: | --- | ---: | ---: | ---: | --- |
| A | | | 3 | 60 | | | | | | |
| B | | | 3 | 60 | | | | | | |


| Condition | Requested model | Returned model version | Attempts per task | Total attempts | pass@1 | 95 percent CI for pass@1 | pass@2 | Input tokens | Output tokens | Dollars spent |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| A | gpt‑5.6‑luna | gpt‑5.6‑luna | 3 | 60 | 0.95 | [0.85, 1.0] | 0.95 | 7146 | 3767 | 0.21 |
| B | gpt‑5.6‑terra | gpt‑5.6‑terra | 3 | 60 | 1.0 | [1.0, 1.0] | 1.0 | 7146 | 4252 | Included in 'A' |



### Memo, no more than 500 words, not counting the table

Address all five items:

1. State the observed ranking by pass@1 point estimate.
2. State whether the uncertainty evidence supports ranking the two conditions.
3. If it does not, include the exact sentence: `The evidence does not support a ranking.`
4. State one external-validity limitation specific to `CS690-Eval20`.
5. State one likely source of variance specific to this experiment, and explain why a rerun, or a classmate's run, gives somewhat different numbers.

Overlapping intervals are not a formal significance test, and you are not asked to run one.

--
The observed ranking by the pass@1 point estimate is straightforward: Condition id B (pass@1 = 1.0) ranks above Condition id A (pass@1 = 0.95). On point estimates alone, Condition B appears to perform perfectly on these 20 tasks, while Condition A shows a small, non‑zero error rate.

However, the uncertainty evidence does not justify treating this small numerical difference as a reliable ranking. Condition A’s 95 percent confidence interval for pass@1 is [0.85, 1.0], which fully overlaps Condition B’s interval of [1.0, 1.0]. Because Condition A’s interval reaches 1.0, the data is consistent with the possibility that both conditions (A and B) have equal performance. The evidence does not support a ranking.

A key external‑validity limitation specific to CS690‑Eval20 is that the dataset consists of short, self‑contained programming puzzles rather than realistic software‑engineering tasks. Real development work involves multi‑file projects, evolving requirements, debugging, integration with existing systems. Performance on these 20 isolated problems may not generalize to how the models behave when maintaining a large codebase, interpreting ambiguous specifications, or collaborating with human developers.

A likely source of variance specific to this experiment is the sampling randomness introduced by temperature = 1.0 and multiple attempts per task. Each attempt is an independent generation, and the pass@k metric depends on the distribution of correct and incorrect attempts. A rerun—or a classmate’s run will produce somewhat different numbers because the model may generate different solutions on each attempt, even with identical prompts and settings. This variance propagates into per‑task pass@k values and as a result, two runs of the same experiment naturally produce slightly different pass@1 and pass@2 estimates.

## Part 5. Reading a published score, 300 to 400 words

Benchmark chosen (HumanEval, MBPP, LiveCodeBench, or SWE-bench): HumanEval


Use the benchmark's primary paper or its official documentation for the task definition. Cite evidence for any contamination, saturation, or current-status claim, and date any current-status source.


### 1. What does it measure?
HumanEval, introduced in Chen et al., 2021, is a benchmark of 164 short Python programming tasks. Each task provides a function signature, a docstring, and hidden unit tests. The benchmark measures whether a model can generate a correct implementation that passes these tests. Its primary metric, pass@k, estimates the probability that at least one of k sampled completions solves the task. HumanEval therefore measures functional correctness on short, isolated programming puzzles with no external dependencies. This is very similar to the conditions test here in this assignment.

### 2. What does it not measure that a software project may depend on?
HumanEval does not measure many skills required in real software engineering. It does not test multi‑file reasoning, debugging, refactoring, architectural design, API comprehension, long‑context reading, or collaboration. It also does not evaluate performance, security, maintainability, or the ability to work with ambiguous or evolving requirements. 

### 3. How can a reported score rise without the underlying model becoming better?
A reported HumanEval score can rise even when the underlying model has not improved. For example, benchmark saturation has occurred: GPT‑4 and later models approach 100% pass@1, leaving little room for meaningful differentiation. Minor changes in sampling temperature, prompt formatting, or test harness implementation can inflate scores. Additionally, prompt overfitting models learning the typical structure of HumanEval tasks, can raise scores without improving general coding ability. Finally, contamination can artificially boost results: HumanEval tasks and solutions appear widely on GitHub, Kaggle and blogs.

### 4. Could the model have seen the answers already?

End with at least one sentence explaining why the published score is not interchangeable with your `CS690-Eval20` result.

Because HumanEval has been public since 2021, the model could have seen the answers already, either directly or indirectly through derivative datasets. This makes it impossible to guarantee that high scores reflect genuine reasoning rather than memorization.

For these reasons, a published HumanEval score is not interchangeable with 'CS690‑Eval20' result, which uses private, unseen tasks, multiple attempts per problem, and bootstrap confidence intervals to measure performance under controlled conditions.



## References
