<script lang="ts">
    import GAME_STATE from '@stores/GAME_STATE';
    import { GameType } from '@typings/gameState';
    import { type IProgressGameState } from '@typings/progress';
    import { TempReceiveEvent } from '@utils/eventsHandlers';
    import { delay } from '@utils/misc';
    import { type Tweened, tweened } from 'svelte/motion';
    import { scale } from 'svelte/transition';
    import { GetRandonNumberKey, PROGRESS_DURATION, PROGRESS_SIZE } from './config/gameConfig';

    const UserSegmentSize: number = 0.5;
    const UserProgress: Tweened<number> = tweened(0);

    let Visible: boolean = false;

    let ProgressState: IProgressGameState = null;

    let IterationState: 'success' | 'fail' | null = null;

    let KeyListener: ReturnType<typeof TempReceiveEvent>;

    //The code above shows the circle progress when the game is active and type is circle progress
    GAME_STATE.subscribe(state => {
        let shouldShow =
            state.active &&
            state.type === GameType.Progress && !ProgressState;
        if (shouldShow) {
            Visible = true;
            initialise();
        } else if (Visible && !shouldShow) {
            Visible = false;
            ProgressState = null;
            KeyListener?.removeListener();
            KeyListener = null;
        }
    });
    

    
    /** This code is responsible for playing the iteration of the minigame.
    * This code should be called when the user presses the spacebar.
    * The code will return a promise that resolves to true if the user has
    * correctly input the key, and false otherwise.
    */ 
    async function playIteration() {
        const duration = ProgressState.duration;
        UserProgress.set(100, {
            duration,
        });

        return new Promise((resolve, _) => {
            let timeout = setTimeout(() => {
                resolve(false);
            }, duration);

            KeyListener = TempReceiveEvent('ui:keydown', (e: KeyboardEvent) => {
                clearTimeout(timeout);

                UserProgress.set($UserProgress, {
                    duration: 0,
                });

                if (e.key === ProgressState.key) {
                    const targetProg= ProgressState.target.progress;
                    const targetSize = ProgressState.target.size

                    const userProg = $UserProgress;

                    if (
                        userProg > targetProg - UserSegmentSize &&
                        userProg < targetSize + targetProg
                    ) {
                        resolve(true);
                    } else {
                        resolve(false);
                        return;
                    }
                } else {
                    resolve(false);
                }
            });
        });
    }


    /** This code is responsible for starting the game.
     * @param iterations The number of iterations to play.
     * @param difficulty The difficulty of the game.
     */
    async function startGame(iterations, difficulty) {
        if (!$GAME_STATE.active) return;

        KeyListener?.removeListener();
        KeyListener = null;

        UserProgress.set(0, {
            duration: 0,
        });

        ProgressState = {
            target: generateTarget(difficulty),
            duration: generateDuration(difficulty),
            key: GetRandonNumberKey(),
        };

        await delay(500)

        const success = await playIteration();
        IterationState = success ? 'success' : 'fail';

        setTimeout(() => {
            IterationState = null
            if (success && iterations > 0) {
                iterations--;
                if (iterations > 0) {
                    startGame(iterations, difficulty);
                } else {
                    GAME_STATE.finish(true);
                    ProgressState = null;
                    return;
                }
            } else {
                GAME_STATE.finish(false);
                ProgressState = null;
                return;
            }
        }, 500);
    }

    /** This code is responsible for generating a duration for a progress bar based on the difficulty.
     */
    function initialise() {
        if (!$GAME_STATE.active || ProgressState) return;

        const { iterations, difficulty } = $GAME_STATE;
        startGame(iterations, difficulty);
    }

    /**
     * Generate a duration for a progress bar based on the difficulty
     * @param difficulty The difficulty should be between 0 and 100.
     */ 
     function generateDuration(difficulty) {
        /** Set the minimum and maximum duration for a progress bar */ 
        const {MIN, MAX} = PROGRESS_DURATION

        /** Calculate the duration based on the difficulty */ 
        let duration =
        MIN + (MAX - MIN) * ((100 - difficulty) / 100);

        /** Make the duration vary by 20% */ 
        const variation = duration * 0.2;
        const randomVariation = Math.random() * variation;
        duration += randomVariation;

        return duration;
    }

    /** Generate a target segment for the given difficulty.
    * The higher the difficulty, the harder the target will be to hit.
    * @param difficulty The difficulty should be between 0 and 100.
    */
    function generateTarget(difficulty) {
        // Make sure the difficulty is between 0 and 100.
        difficulty = difficulty >= 100 ? 99 : difficulty <= 0 ? 5 : difficulty;

                // Calculate the target size based on the difficulty.
        const { MAX } = PROGRESS_SIZE
        const size = MAX - (difficulty / 100) * MAX;

        /**
         * The target should start atleastr 40% into the progress bar so the user has a chance to know.
         */
        const minProgress = 30;

        /**
         * The target should end before 100% - size so the whole target is visible.
         */
        const maxProgress = 100 - size;


        const progress = Math.random() * (maxProgress - minProgress) + minProgress;

        return {
            size,
            progress,
        };
    }
</script>

{#if Visible}
<div
    transition:scale
    class=" primary-shadow center-x bottom-[5vh] w-[25vw] h-[0.5vw] bg-primary-50"
>
    <div
        class="h-[2.5vw] aspect-square absolute grid place-items-center center-y secondary-shadow bg-primary-50 -translate-x-[130%]"
    >
        <p class="text-shadow absolute font-bold text-[2vw]">{ProgressState.key}</p>
    </div>

    <div
        style="left: {$UserProgress}%; width: {UserSegmentSize}vw"
        class="h-[1vw] center-y z-[10] absolute origin-center transition-colors duration-100 {IterationState ===
        'success'
            ? 'glow-success bg-success'
            : IterationState === 'fail'
            ? 'glow-fail bg-fail'
            : 'bg-accent glow-accent'}"
    />

    {#if ProgressState}
        {@const { size, progress } = ProgressState.target}
        <div
            style="left: {progress}%; width: {size}%"
            class="h-[1vw] center-y absolute origin-center bg-primary z-0 target-segment"
        />
    {/if}
</div>
{/if}



<style>
        .target-segment {
        filter: drop-shadow(0 0 0.1vw black);
        transition: all 0.1s ease-in-out;
    }
</style>