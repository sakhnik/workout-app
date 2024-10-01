<script>
    import { onMount } from 'svelte';
    import { register, init, isLoading, locale, getLocaleFromNavigator, _ } from "svelte-i18n";
    import "flag-icons/css/flag-icons.min.css";


    register('en', () => fetch('/i18n/en.json').then(res => res.json()));
    register('uk', () => fetch('/i18n/uk.json').then(res => res.json()));

    init({
        fallbackLocale: 'uk',
        initialLocale: getLocaleFromNavigator(),
    });

    let sets = 10;
    let currentSet = 1;
    let currentExercise = 0;
    let isResting = true; // Initially, we rest before exercise
    let workoutStarted = false;
    let workoutFinished = false;
    let timer;
    let startTime;
    let restStartTime;
    let totalRestTime = 0;
    let timerTime = 0;
    let estimatedFinishTime = '';

    const exercises = [
        { name: 'Pullups', reps: 10 },
        { name: 'Pushups', reps: 20 },
        { name: 'Crunches', reps: 30 }
    ];

    let stepsDone = 0;

    // Start the workout and calculate the estimated finish time
    function startWorkout() {
        workoutStarted = true;
        startTime = new Date();
        totalRestTime = 0;
        startRest();
    }

    // Calculate the total workout duration in seconds based on sets, exercises, and rest
    function calculateEstimatedFinishTime(stage) {
        if (stepsDone === 0) {
            return;
        }
        const now = new Date().getTime();
        const elapsedTime = now - startTime;
        const rate = elapsedTime / stepsDone;
        const stepsTodo = 2 * ((sets - currentSet) * exercises.length + (exercises.length - currentExercise)) - stage;
        const finishDate = new Date(Math.floor(now + (stepsTodo * rate)));
        estimatedFinishTime = finishDate.toLocaleTimeString();
    }

    // Show upcoming exercise after rest
    function showExercise() {
        totalRestTime += new Date().getTime() - restStartTime;
        isResting = false;
        calculateEstimatedFinishTime(1);
        ++stepsDone;
        startTimer();
    }

    // Start the rest period
    function startRest() {
        isResting = true;
        restStartTime = new Date().getTime();
        calculateEstimatedFinishTime(0);
        ++stepsDone;
        startTimer();
    }

    function startTimer() {
        clearInterval(timer);
        let start = new Date().getTime();
        timerTime = 0;

        timer = setInterval(() => {
            timerTime = Math.floor((new Date().getTime() - start) / 1000);
        }, 1000);
    }

    function formatSeconds(seconds) {
        const minutes = Math.floor(seconds / 60);
        if (minutes === 0) {
            return `${seconds}`;
        }
        
        const remainingSeconds = seconds % 60;
        const formattedSeconds = remainingSeconds.toString().padStart(2, '0');
        return `${minutes}:${formattedSeconds}`;
    }

    function formatTime(seconds) {
        const minutes = Math.floor(seconds / 60);
        const remainingSeconds = seconds % 60;
        const formattedSeconds = remainingSeconds.toString().padStart(2, '0');
        return `${minutes}:${formattedSeconds}`;
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
    .language-buttons {
        position: absolute;
        top: 10px;
        right: 10px;
    }

    button {
        margin-left: 10px;
        padding: 5px 10px;
        font-size: 16px;
    }
</style>

{#if $isLoading}
Please wait…
{:else}

{#if !workoutStarted && !workoutFinished}
    <!-- Initial Screen to set number of sets -->
    <div class="container">
        <h1>{$_('start.sets?')}</h1>
        <input type="number" bind:value={sets} min="1" max="50">
        <button class="button" on:click={startWorkout}>{$_('start.workout')}</button>
    </div>
{:else if workoutStarted && !workoutFinished}
    <!-- Workout Screen -->
    <div class="container">
        {#if isResting}
            <!-- Display rest and the upcoming exercise -->
            <div class="exercise rest">
                {$_('workout.resting')}: {$_(exercises[currentExercise].name)}
            </div>
        {:else}
            <!-- Display current exercise -->
            <div class="exercise">
                {$_(exercises[currentExercise].name)}: x{exercises[currentExercise].reps}
            </div>
        {/if}
        <div>{$_('workout.set')} {currentSet}/{sets}, {$_('workout.exercise')} {currentExercise + 1}/{exercises.length}</div>
        <div>{$_('workout.time')}: {formatSeconds(timerTime)} {$_('workout.s')}</div>

        <!-- Button group for next/previous -->
        <div class="button-group">
            <button class="button" on:click={previousExercise} disabled={currentExercise === 0 || isResting}>{$_('workout.previous')}</button>
            <button class="button" on:click={nextExercise}>{$_('workout.next')}</button>
        </div>

        <!-- Estimated finish time -->
        <div class="estimated-time">
            {$_('workout.eta')}: {estimatedFinishTime}
        </div>
    </div>
{:else}
    <!-- Summary Screen -->
    <div class="container">
        <h1>{$_('summary.complete')}</h1>
        <p>{$_('summary.total-time')}: {formatTime(Math.floor((new Date() - startTime) / 1000))}</p>
        <p>{$_('summary.rest-time')}: {formatTime(Math.floor(totalRestTime / 1000))} {Math.floor(100 * totalRestTime / (new Date() - startTime))}%</p>
    </div>
{/if}

<div class="language-buttons">
    <button on:click={() => locale.set('en')}><span class="fi fi-gb"></span></button>
    <button on:click={() => locale.set('uk')}><span class="fi fi-ua"></span></button>
</div>

{/if}
