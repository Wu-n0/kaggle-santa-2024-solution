# Santa 2024 - Word Unscrambling Challenge Solution

## Overview

This repository contains my solution for the Kaggle Santa 2024 competition. The challenge involved rearranging scrambled words in Christmas stories to minimize their perplexity score, essentially putting the words back in an order that makes the most sense.

All computations were performed on an NVIDIA RTX 4070 Super GPU, with each perplexity calculation taking approximately 1.15 seconds. This meant careful optimization of the approach was necessary to efficiently explore the solution space.

## Solution Approach

I implemented Simulated Annealing (SA) to solve this problem. SA is a probabilistic technique for finding good solutions to optimization problems where the search space is too large for exhaustive search.

### Simulated Annealing Implementation Details

My SA implementation is designed specifically for word reordering to minimize perplexity:

```python
class SimulatedAnnealing:
    def __init__(self, Tmax, Tmin, nsteps, nsteps_per_T, log_freq, random_state, cooling, k, batch_size):
        # Initialize parameters
        # ...
```

#### Key Parameters

- `Tmax`: Starting temperature - controls initial acceptance probability of worse solutions
- `Tmin`: Ending temperature - defines when the algorithm starts to be more selective
- `nsteps`: Number of temperature steps in the cooling schedule
- `nsteps_per_T`: Number of iterations at each temperature level
- `cooling`: Type of cooling schedule (linear, exponential, or logarithmic)
- `k`: Scaling factor for acceptance probability calculation
- `batch_size`: Batch size for perplexity calculations to optimize GPU usage

*Note: These parameters were tuned differently for each paragraph based on its complexity and length.*

#### Neighbor Generation Strategies

I implemented three different neighbor generation methods that are randomly selected during each iteration:

1. **Swap Method**: Randomly selects two words and swaps their positions
   ```python
   i, j = random.sample(range(len(neighbor)), 2)
   neighbor[i], neighbor[j] = neighbor[j], neighbor[i]
   ```

2. **Single Word Shift**: Extracts a single word and inserts it at a new position
   ```python
   extract, insert = random.sample(range(len(shift) - 1), 2)
   shift_words = shift[extract : extract + 1]
   shift = shift[:extract] + shift[extract + 1 :]
   shift = shift[:insert] + shift_words + shift[insert:]
   ```

3. **Adjacent Words Shift**: Moves two adjacent words to a new position
   ```python
   extract = random.randint(0, len(shift) - 2)
   shift_words = shift[extract : extract + 2]
   shift = shift[:extract] + shift[extract + 2 :]
   insert = random.randint(0, len(shift))
   shift = shift[:insert] + shift_words + shift[insert:]
   ```

#### Anti-Stagnation Mechanism

To escape local optima, I implemented a "temperature restart" mechanism that temporarily increases the temperature when no improvement is found after 10,000 iterations:

```python
if no_improvement_count >= 10000:
    restart_temp = temperature + random.uniform(0, (self.Tmax/2 - temperature))
    temperature = restart_temp
    no_improvement_count = 0  # Reset counter after restart
```

This allows the algorithm to escape deep local minima by temporarily accepting more worse solutions, before gradually cooling back down to find better optima.

## Algorithm Operation

The core solving logic follows this process:

1. **Initialization**: Start with the scrambled text and calculate its perplexity score
2. **Main Loop**: For each temperature step:
   - Perform `nsteps_per_T` iterations (typically 1000)
   - In each iteration:
     - Generate a neighbor solution using one of the three methods
     - Calculate its perplexity score
     - Accept or reject based on the Metropolis criterion
     - Update best solution if improved
   - Apply temperature restart if stuck in local optima
   - Reduce temperature according to cooling schedule

3. **Acceptance Probability**: The probability of accepting a worse solution is:
   ```python
   math.exp(self.k * (current_energy - new_energy) / temperature)
   ```
   This means:
   - Better solutions are always accepted
   - Slightly worse solutions have a higher chance of acceptance
   - Very poor solutions have a lower chance of acceptance
   - As temperature decreases, the algorithm becomes more selective

4. **Early Stopping**: If a solution with perplexity below a target threshold is found, the algorithm stops

## Results by Paragraph

### Paragraph 0 
- Since this paragraph only had 10 words, the community was able to solve it via brute force (10! possible arrangements)
- Correct sentence: "reindeer mistletoe elf gingerbread family advent scrooge chimney fireplace ornament"
- Score: 469.5

### Paragraph 1
- Reached the optimal solution in a single SA run
- Required approximately 30,000 iterations
- Score: 424

### Paragraph 2
- Reached the optimal solution in a single SA run
- Required approximately 50,000 iterations
- Score: 299

### Paragraph 3
- Difficult paragraph with harsh local optima
- Ran over 1,500,000 iterations but never reached the global optimum
- According to top solutions, most successful approaches fixed the first word and then ran SA on the remaining words
- My Score: 197.5
- Optimal Score: 191.5

### Paragraph 4
- Ran approximately 1,500,000 iterations
- Got stuck in local optima despite multiple restart attempts
- My Score: 70
- Optimal Score: 67.5

### Paragraph 5 
- By far the most challenging paragraph with 100 words and harsh local optima
- Ran through 2,000,000 iterations
- My Score: 31
- Optimal Score: 28.5

## Score Summary

| Paragraph | My Score | Optimal Score | Difference |
|-----------|----------|---------------|------------|
| 0  | 469.5 | 469.5 | 0 |
| 1 | 424.0 | 424.0 | 0 |
| 2 | 299.0 | 299.0 | 0 |
| 3 | 197.5 | 191.5 | 6.0 |
| 4 | 70.0 | 67.5 | 2.5 |
| 5 | 31.0 | 28.5 | 2.5 |
| **Total** | **248.6** | **246.8** | **1.8** |

## Lessons Learned

Different paragraphs required custom parameter tuning based on length and complexity. Longer paragraphs needed more iterations and higher initial temperatures. The temperature restart mechanism proved essential for escaping local optima, particularly for paragraphs 3-5. For paragraph 3, fixing the first word and optimizing the rest was more effective than treating all words equally.
