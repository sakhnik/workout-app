<script>
    import { onMount } from 'svelte';

    let sets = 10;
    let currentSet = 1;
    let currentExercise = 0;
    let restTime = 60; // Rest time in seconds
    let exerciseTime = 30; // Each exercise duration in seconds
    let isResting = true; // Initially, we rest before exercise
    let workoutStarted = false;
    let workoutFinished = false;
    let timer;
    let startTime;
    let elapsedTime = 0;
    let estimatedFinishTime = 0; // Estimated time in seconds

    const exercises = [
        { name: 'Pullups', reps: 10 },
        { name: 'Pushups', reps: 20 },
        { name: 'Crunches', reps: 30 }
    ];

    // Start the workout and calculate the estimated finish time
    function startWorkout() {
        workoutStarted = true;
        startTime = new Date();
        calculateEstimatedFinishTime();
        startRest();
    }

    // Calculate the total workout duration in seconds based on sets, exercises, and rest
    function calculateEstimatedFinishTime() {
        const totalExercises = sets * exercises.length;
        const totalRestTime = totalExercises * restTime;
        const totalExerciseTime = totalExercises * exerciseTime;
        estimatedFinishTime = Math.floor((new Date().getTime() + (totalRestTime + totalExerciseTime) * 1000) / 1000);
    }

    // Show upcoming exercise after rest
    function showExercise() {
        isResting = false;
        startTimer(exerciseTime); // Exercise lasts exerciseTime seconds
    }

    // Start the rest period
    function startRest() {
        isResting = true;
        startTimer(restTime); // Rest lasts restTime seconds
    }

    // Start a countdown timer and switch between rest and exercise
    function startTimer(duration) {
        clearInterval(timer);
        elapsedTime = 0;

        timer = setInterval(() => {
            elapsedTime += 1;
            if (elapsedTime >= duration) {
                nextExercise();
            }
        }, 1000);
    }

    // Move to the next exercise or set
    function nextExercise() {
        clearInterval(timer);

        if (isResting) {
            showExercise();
        } else {
            // If all exercises are done in a set, move to the next set
            if (currentExercise < exercises.length - 1) {
                currentExercise += 1;
                startRest();
            } else {
                currentExercise = 0;
                currentSet += 1;

                if (currentSet > sets) {
                    finishWorkout();
                } else {
                    startRest();
                }
            }
        }
    }

    // Move to the previous exercise (cannot go back during rest)
    function previousExercise() {
        if (!isResting && currentExercise > 0) {
            currentExercise -= 1;
            showExercise();
        }
    }

    // Finish the workout and display summary
    function finishWorkout() {
        clearInterval(timer);
        workoutFinished = true;
    }

    // Handle key inputs for 'n' (next) and 'p' (previous)
    function handleKeyDown(event) {
        if (event.key === 'n') {
            nextExercise();
        } else if (event.key === 'p') {
            previousExercise();
        }
    }

	// Detect reload and prompt user
    function handleBeforeUnload(event) {
        if (!workoutFinished) {
            const confirmationMessage = "Are you sure you want to abandon your workout?";
            event.returnValue = confirmationMessage; // Standard for most browsers
            return confirmationMessage;             // Some browsers require this
        }
    }

    onMount(() => {
        window.addEventListener('keydown', handleKeyDown);
        window.addEventListener('beforeunload', handleBeforeUnload);

        return () => {
            window.removeEventListener('keydown', handleKeyDown);
            window.removeEventListener('beforeunload', handleBeforeUnload);
        };
    });

    // Format estimated finish time
    function formatEstimatedFinishTime() {
        const finishDate = new Date(estimatedFinishTime * 1000);
        return finishDate.toLocaleTimeString();
    }
</script>

<style>
    .container {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        height: 100vh;
    }
    .button {
        background-color: #3498db;
        border: none;
        padding: 10px 20px;
        color: white;
        cursor: pointer;
        margin-top: 20px;
    }
    .exercise {
        font-size: 2rem;
        font-weight: bold;
        margin-top: 20px;
    }
    .rest {
        color: red;
    }
    .button-group {
        display: flex;
        gap: 10px;
        margin-top: 20px;
    }
</style>

{#if !workoutStarted && !workoutFinished}
    <!-- Initial Screen to set number of sets -->
    <div class="container">
        <h1>How many sets?</h1>
        <input type="number" bind:value={sets} min="1" max="50">
        <button class="button" on:click={startWorkout}>Start Workout</button>
    </div>
{:else if workoutStarted && !workoutFinished}
    <!-- Workout Screen -->
    <div class="container">
        {#if isResting}
            <!-- Display rest and the upcoming exercise -->
            <div class="exercise rest">
                Rest for {restTime - elapsedTime} seconds
            </div>
            <div>Upcoming: {exercises[currentExercise].name}</div>
            <div>Set {currentSet}/{sets}, Exercise {currentExercise + 1}/{exercises.length}</div>
        {:else}
            <!-- Display current exercise -->
            <div class="exercise">
                {exercises[currentExercise].name}: {exercises[currentExercise].reps} reps
            </div>
            <div>Set {currentSet}/{sets}, Exercise {currentExercise + 1}/{exercises.length}</div>
            <div>Time remaining: {exerciseTime - elapsedTime} seconds</div>
        {/if}

        <!-- Button group for next/previous -->
        <div class="button-group">
            <button class="button" on:click={previousExercise} disabled={currentExercise === 0 || isResting}>Previous</button>
            <button class="button" on:click={nextExercise}>Next</button>
        </div>

        <!-- Estimated finish time -->
        <div class="estimated-time">
            Estimated finish time: {formatEstimatedFinishTime()}
        </div>
    </div>
{:else}
    <!-- Summary Screen -->
    <div class="container">
        <h1>Workout Complete!</h1>
        <p>Total time: {Math.floor((new Date() - startTime) / 1000)} seconds</p>
    </div>
{/if}
