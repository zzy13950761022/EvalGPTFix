# EvalGPTFix: Evaluating ChatGPT for Automated Program Repair

EvalGPTFix is a benchmark and evaluation framework for assessing the effectiveness of ChatGPT (GPT-3.5-turbo) on **automated program repair**. It uses **AtCoder programming contest problems** as the dataset and evaluates repair performance across multiple prompt strategies and interaction modes.

The evaluation is structured around four research questions (RQs), examining the impact of prompt engineering, error feedback, bug localization, and multi-turn dialogue on LLM-based program repair.

## Research Questions

| RQ | Focus | Description |
|----|-------|-------------|
| **RQ1** | Basic Effectiveness | Can ChatGPT fix buggy programs when given only the buggy code? |
| **RQ2** | Prompt Engineering | How do different prompt strategies (problem description, error information, bug location) affect repair success? |
| **RQ3** | Multi-Turn Dialogue | Can iterative dialogue with execution feedback improve repair outcomes? |
| **RQ4** | Self-Repair | Can ChatGPT repair its own generated solutions through self-feedback? |

## Dataset

The benchmark uses problems from the **AtCoder** programming contest platform. Each buggy program is a Java class (`Main.java`) that fails one or more test cases.

### Bug Types

| Type | Description |
|------|-------------|
| **CE** | Compilation Error — the program fails to compile. |
| **WA** | Wrong Answer — the program compiles but produces incorrect output. |
| **TLE** | Time Limit Exceeded — the program runs longer than the allowed time limit. |
| **MLE** | Memory Limit Exceeded — the program exceeds the memory limit. |
| **RE** | Runtime Error — the program crashes during execution. |

### Data Format

The dataset (`data/latest_contests_new.csv`) contains the following fields:

```
username, task_id, language, buggy_code, [additional metadata]
```

Each task is identified by a contest/problem pair (e.g., `abc297/G`), and the corresponding test cases are stored in `AtCoderTestCasesOfficial/`.

## Prompt Strategies (RQ2)

Three prompt strategies are evaluated, each providing progressively more information to the model:

| Strategy | Information Provided | Prompt Key |
|----------|---------------------|------------|
| **Problem-Aware** | Buggy code + full problem description | `query_with_problem` |
| **Error-Info** | Buggy code + error type + failing input + expected output | `query_with_error_info` |
| **Bug-Location** | Buggy code with `//bug` comment marking the bug location | `query_with_bug_location` |

## Multi-Turn Dialogue (RQ3)

In RQ3, when a fix attempt fails, the execution result (error type, input, expected/actual output) is fed back to ChatGPT for another repair attempt. This iterative process continues for up to 5 rounds or until the fix passes all test cases.

```
Initial Fix → Execute → [Failure] → Generate Error Feedback → Attempt Fix → ...
                                                  ↓
                                            [Success] → Stop
```

## Evaluation Pipeline

```
AtCoder Problem → Buggy Java Code → ChatGPT Fix → Compile (javac) → Run Tests → Result
                                                                        ↓
                                                            CE / WA / TLE / MLE / RE / AC
```

Each generated fix is:
1. Written to `Main.java`
2. Compiled with `javac`
3. Run against official AtCoder test cases
4. Classified as one of: **AC** (Accepted), **CE**, **WA**, **TLE**, **MLE**, or **RE**

## Repository Structure

```
EvalGPTFix/
├── rq.py                              # Main evaluation script (all four RQs)
├── run_testcase.py                    # Java compilation and test execution engine
├── run_all_testcases.py               # Batch test execution across all solutions
│
├── data/                              # Dataset files
│   ├── latest_contests.csv            # AtCoder problem metadata
│   ├── latest_contests_new.csv        # Main dataset with buggy code
│   ├── latest_contests_with_bug_location.csv
│   ├── latest_contests_with_bug_location_new.csv
│   ├── Atcoder_data.csv
│   └── new_atcoder_data.csv
│
├── results/                           # Experimental results (~848 MB)
│   ├── results_RQ1/                   # RQ1: basic ChatGPT fix results
│   ├── results_RQ2_error_info/        # RQ2: error-info prompt results
│   ├── results_RQ2_bug_location/      # RQ2: bug-location prompt results
│   ├── RQ2_problems/                  # RQ2: problem-aware prompt results
│   └── results_RQ3/                   # RQ3: multi-turn dialogue results
│
├── problems/                          # AtCoder problem descriptions
├── AtCoderTestCasesOfficial/          # Official test cases (input/output pairs)
├── error_info/                        # Per-bug error trace data
├── initial_problem_solutions/         # GPT-generated initial solutions (RQ4)
├── gpt_solutions/                     # Best-of-10 GPT solutions (RQ4)
└── results_RQ4_final/                 # RQ4: self-repair results
```

## Usage

### Prerequisites

- Python 3.8+
- Java 8+ (JDK)
- OpenAI API key

### Step 1 — Configure API Key

Set your OpenAI API key in `rq.py`:

```python
openai.api_key = 'your-api-key'
```

### Step 2 — Run Evaluations

```bash
# RQ1: Basic ChatGPT effectiveness
python -c "from rq import query_only_with_bug; query_only_with_bug()"

# RQ2: Prompt engineering (problem-aware)
python -c "from rq import query_with_problem; query_with_problem()"

# RQ2: Prompt engineering (error-info)
python -c "from rq import query_with_error_info; query_with_error_info()"

# RQ2: Prompt engineering (bug-location)
python -c "from rq import query_with_bug_location; query_with_bug_location()"

# RQ3: Multi-turn dialogue repair
python -c "from rq import perform_more_dialogues; perform_more_dialogues()"

# RQ4: Self-repair
python -c "from rq import self_repair; self_repair()"
```

### Step 3 — Analyze Results

Results are saved in the `results/` directory, organized by RQ and bug ID. Each result file is named with the format:

```
{attempt_number}_{result_type}.txt
```

Where `result_type` is one of: `AC`, `CE`, `WA`, `TLE`, `MLE`, `RE`.

## Test Execution Engine

`run_testcase.py` provides the core evaluation infrastructure:

- **`run_test_case(code, task, memory)`** — Compiles and runs a single Java program against all test cases for a given task. Returns the first failure encountered.
- **`run_all_test_case(code, task, memory)`** — Runs all test cases and returns per-case results, used for selecting the best solution.

The engine:
1. Writes the code to `Main.java`
2. Compiles with `javac -J-Dfile.encoding=UTF8`
3. Executes with `java -Xmx{memory}m Main`, piping each test input via stdin
4. Compares actual output against expected output
5. Cleans up generated `.class` and `Main.java` files

## Technical Details

### Model Configuration

- **Model**: GPT-3.5-turbo (`gpt-3.5-turbo`)
- **API**: OpenAI ChatCompletion API
- **Temperature**: default (not specified)
- **Max Context**: 4097 tokens (enforced in RQ3/RQ4 dialogue mode)

### Error Handling

The framework handles several API-level failures:
- Transient API errors: automatic retry with `while not success` loop
- Context length exceeded: dialogue messages are pruned (oldest pair removed) and retried
- Rate limiting: handled by OpenAI library defaults

### Code Extraction

LLM responses are parsed by extracting the content within the first markdown code block (` ``` `). This handles the common case where ChatGPT wraps code in explanatory text.

## Research Applications

EvalGPTFix is designed to support research in:

- **LLM-based program repair** — evaluating how effectively LLMs can fix buggy code across different bug types.
- **Prompt engineering for repair** — studying how different prompt designs (problem context, error traces, bug localization) affect repair success rates.
- **Iterative repair with feedback** — analyzing whether multi-turn dialogue with execution feedback improves repair outcomes.
- **Self-repair** — investigating whether LLMs can autonomously fix their own generated code.
- **Repair on competitive programming** — using AtCoder problems as a standardized, well-tested benchmark for program repair evaluation.

## Citation

If you use EvalGPTFix in your research, please cite the original work.

## License

This project is provided for research purposes.