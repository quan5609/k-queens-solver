<template>
    <div class="flex flex-wrap -mx-4">
        <div class="w-full px-4 md:w-8/12">
            <!-- ref="boardContainerRef" -->
            <!-- Main Chess's board -->
            <ChessBoard
                class="mx-auto mt-16 md:mt-20"
                :showLegends="true"
                :kNumber="kNumber"
                :edgeLength="edgeLength"
            />

            <!-- Game actions -->
            <div class="mt-10 space-y-3 ">
                <div class="text-center space-x-2">
                    <span class="text-lg font-semibold">K</span>
                    <span> = </span>
                    <input
                        class="w-16 px-3 py-2 leading-tight text-center border border-blue-500 rounded shadow appearance-none focus:outline-none focus:shadow-outline"
                        type="text"
                        v-model="kNumberComputed"
                    />
                </div>

                <div class="text-center space-x-2">
                    <button
                        class="shadow-md btn btn-primary"
                        @click="randomize"
                    >Randomize</button>

                    <button
                        class="shadow-md btn btn-primary"
                        @click="reset"
                    >Reset</button>
                </div>

                <div class="text-center space-x-2">
                    <button
                        class="shadow-md btn btn-success"
                        @click="solve"
                    >Solve</button>
                </div>

                <div class="flex items-center w-full pb-4 mx-auto sm:w-8/12 xl:w-6/12 space-x-4">
                    <span class="font-semibold">Speed: </span>
                    <VueSlider
                        class="flex-grow"
                        v-model="speedComputed"
                        :data="['0.25x', '0.5x', '1x', '2x', '3x', '4x']"
                        :marks="true"
                        :absorb="true"
                        :lazy="true"
                    />
                    <font-awesome-icon
                        :icon="['fas', 'running']"
                        size="lg"
                        class="text-blue-600"
                    ></font-awesome-icon>
                </div>
            </div>
        </div>

        <!-- List of steps -->
        <div class="w-full px-4 md:w-4/12">
            <ListMoves
                :steps="steps"
                @moveRefsChanged="(refs) => moveRefs = refs"
            />
        </div>
    </div>
</template>

<script lang="ts">
import {
    defineComponent,
    ref,
    computed,
    watch,
    reactive,
    onMounted,
    watchEffect,
    onBeforeUnmount,
} from "@vue/composition-api";
import VueSlider from "vue-slider-component";
import "vue-slider-component/theme/default.css";
import gsap from "gsap";
import axios, { AxiosResponse } from "axios";
import { bus } from "@/main";
import * as interfaces from "@/interfaces/interfaces";
import ChessBoard from "@/components/ChessBoard/ChessBoard.vue";
import ListMoves from "@/components/ListMoves/ListMoves.vue";

export default defineComponent({
    name: "MinConflict",

    components: {
        VueSlider,
        ChessBoard,
        ListMoves,
    },

    setup() {
        //---------------- Essentials ----------------
        const timelines = reactive({
            queens: gsap.timeline({
                default: {
                    ease: "none",
                    force3D: false,
                },
            }),
            moves: gsap.timeline({
                default: {
                    ease: "none",
                    force3D: false,
                },
            }),
        });

        const kNumber = ref<number>(4);
        const kNumberComputed = computed({
            get: () => kNumber.value + "",
            set: (value) => {
                kNumber.value = parseInt(value);
            },
        });

        const edgeLength = ref<number>(4);

        //---------------- Default's state ----------------
        const defaultY = 1;
        function generateDefaultState(k: number) {
            const result: interfaces.Position[] = [];
            for (let i = 1; i <= k; ++i) {
                result.push({
                    x: i,
                    y: defaultY,
                });
            }

            return result;
        }

        //---------------- Solving's logics ----------------
        const currentState = ref<interfaces.Position[]>(generateDefaultState(kNumber.value));

        const steps = ref<interfaces.Step[]>([]);

        //---------------- Animation's speed ----------------
        const baseDuration = 0.5;
        const speed = ref<number>(1);
        const speedComputed = computed({
            get: () => {
                return speed.value + "x";
            },
            set: (value) => {
                speed.value = parseFloat(value);
            },
        });

        //---------------- Template's references ----------------
        //        const boardContainerRef = ref<Element>(null as any);
        //        const boardContainerWidth = ref<number>(0);
        //        const refreshScrollArea: number = setInterval(() => {
        //            boardContainerWidth.value = boardContainerRef.value.clientWidth;
        //        }, 2000);

        const queenRefs = ref<Element[]>([]);
        bus.$on("queenRefs", (refs: Element[]) => (queenRefs.value = refs));

        const moveRefs = ref<Element[]>([]);

        //---------------- Functionalities ----------------
        function resetMoves() {
            if (moveRefs.value.length !== 0) {
                gsap.to(moveRefs.value, {
                    opacity: 0,
                    duration: 0.25,
                    onComplete: () => {
                        steps.value = [];
                    },
                });
            }
        }

        function randomize() {
            currentState.value = [];

            for (let col = 1; col <= kNumber.value; ++col) {
                const row = Math.floor(Math.random() * (kNumber.value - 1)) + 1;

                gsap.to(queenRefs.value[col - 1], {
                    // x: `${col * edgeLength.value}rem`,
                    y: `${(row - defaultY) * edgeLength.value}rem`,
                    ease: "none",
                    duration: 0.5,
                    force3D: false,
                });

                currentState.value.push({
                    x: col,
                    y: row,
                });
            }

            resetMoves();
        }

        function reset() {
            currentState.value = generateDefaultState(kNumber.value);

            gsap.to(queenRefs.value, {
                // x: 0,
                y: 0,
                ease: "none",
                duration: 0.25,
                force3D: false,
            });

            resetMoves();
        }

        async function solve() {
            const postData: interfaces.MinConflictPostRequest = {
                k: kNumber.value,
                initState: currentState.value.map((pos) => pos.y - 1),
                iteration: 1000 * kNumber.value,
            };

            try {
                const response: AxiosResponse<interfaces.MinConflictPostResponse> = await axios.post(
                    "/min-conflict",
                    postData
                );

                steps.value = response.data.steps;
            } catch (err) {
                console.error(err);
            }
        }

        //---------------- Watchers ----------------
        // Reset 'currentState' whenever 'kNumber' change
        watch(kNumber, (curr) => {
            console.log("[watch] 'kNumber' changed...");

            switch (true) {
                case curr <= 4: {
                    edgeLength.value = 4;
                    break;
                }
                case curr > 4 && curr <= 8: {
                    edgeLength.value = 3.5;
                    break;
                }
                case curr > 8 && curr <= 11: {
                    edgeLength.value = 3;
                    break;
                }
                case curr > 11 && curr <= 14: {
                    edgeLength.value = 2.5;
                    break;
                }
                case curr > 14 && curr <= 17: {
                    edgeLength.value = 2;
                    break;
                }
                case curr > 17 && curr <= 20: {
                    edgeLength.value = 1.75;
                    break;
                }
                default: {
                    edgeLength.value = 1.25;
                    break;
                }
            }

            reset();
        });

        // Animate whenever solved a solution
        watch(steps, (curr) => {
            console.log("[watch] 'steps' changed...");
            curr.forEach((step) => {
                timelines.queens.to(queenRefs.value[step.choice.x - 1], {
                    //x: `${step.choice.x * edgeLength.value}rem`,
                    y: `${(step.choice.y - defaultY) * edgeLength.value}rem`,
                    duration: baseDuration,
                });
            });
        });

        watch(moveRefs, (curr) => {
            console.log("[watch] 'moveRefs' changed...");
            curr.forEach((element) => {
                timelines.moves.to(element, {
                    opacity: 1,
                    duration: baseDuration,
                });
            });
        });

        // Change animation's speed
        watch(speed, (curr) => {
            console.log("[watch] 'speed' changed...");
            timelines.queens.timeScale(curr);
            timelines.moves.timeScale(curr);
        });

        //        watch(boardContainerWidth, (curr) => {});
        //
        //        onBeforeUnmount(() => {
        //            clearInterval(refreshScrollArea);
        //        });

        return {
            kNumber,
            kNumberComputed,
            edgeLength,
            steps,
            speedComputed,
            //            boardContainerRef,
            moveRefs,
            randomize,
            reset,
            solve,
        };
    },
});
</script>
