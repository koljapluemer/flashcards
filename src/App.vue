<script setup>
// import words from "./words.js";
import sentences from "./new_sentences.js";
import { ref } from "vue";
import * as ebisu from "ebisu-js";

let randomExercise = ref([]);
let exerciseBank = ref([]);
let unitsPracticedToday = 0;
let unitsPracticedYesterday = 0;
// see if exerciseBank is in localStorage, if so, load it,  if not, set it to the imported exercises

if (!localStorage.getItem("exerciseBank")) {
  exerciseBank.value = sentences;
} else {
  // if it is in localStorage, set the exerciseBank to the localStorage value
  exerciseBank.value = JSON.parse(localStorage.getItem("exerciseBank"));
}

// for every exercise in the exerciseBank, add properties: model {} and seenAt timestamp (if they don't exist)
exerciseBank.value.forEach((exercise) => {
  if (!exercise.model) {
    // using 2h as half life (have to remember that unit is just h now)
    exercise.model = ebisu.defaultModel(2);
  }
  if (!exercise.seenAt) {
    exercise.seenAt = new Date().getTime() / 1000;
  }
  if (!exercise.stats) {
    exercise.stats = [];
  }
});

// same with localStorage stats
let stats = {};
if (!localStorage.getItem("stats")) {
  stats = {
    counter: 0,
  };
} else {
  stats = JSON.parse(localStorage.getItem("stats"));
}

let isRevealed = ref(false);

let valueEase = ref(null);
let valueCorrect = ref(null);
let valueAnki = ref(null);

function getNextExercise() {
  // get exercise with highest probability of being forgotten
  // use: ebisu.predictRecall(model, elapsed, true):
  // and return the exercise with the lowest probability
  // make sure elapsed is in hours
  exerciseBank.value.forEach((exercise) => {
    exercise.probability = ebisu.predictRecall(
      exercise.model,
      (new Date().getTime() / 1000 - exercise.seenAt) / 3600,
      true
    );
  });
  exerciseBank.value.sort((a, b) => a.probability - b.probability);
  randomExercise.value = exerciseBank.value[0];
}

getNextExercise();

function evaluateScore(score) {
  const model = randomExercise.value.model;
  const successes = score;
  const total = 2;
  const elapsed =
    (new Date().getTime() / 1000 - randomExercise.value.seenAt) / 3600;
  const newModel = ebisu.updateRecall(model, successes, total, elapsed);
  randomExercise.value.model = newModel;
  randomExercise.value.seenAt = new Date().getTime() / 1000;
  isRevealed.value = false;
  randomExercise.value.stats.push({
    score,
    time: new Date().getTime() / 1000,
  });
  console.log("Exercise Data after evaluation: ", randomExercise.value);
  // save exerciseBank to localStorage
  localStorage.setItem("exerciseBank", JSON.stringify(exerciseBank.value));
  getNextExercise();
}
</script>

<template>
  <small> Practiced {{ stats.counter }} times so far </small>
  <div class="" v-if="!randomExercise.en">nothing more to practice.</div>

  <div
    class="card bg-gray-600 shadow-xl m-4 flex flex-col items-center w-full max-w-screen-xl"
    v-else
  >
    <div class="card-body">
      <h2 class="card-title my-2 text-2xl">{{ randomExercise.en }}</h2>
      <div :class="{ hidden: !isRevealed }">
        <div class="flex flex-col gap-2">
          <p class="my-2 text-4xl">
            {{ randomExercise.ar }}
          </p>
          <p class="my-2 text-xl">
            {{ randomExercise.tr }}
          </p>
          <div class="div">
            <a
              target="_blank"
              :href="
                'https://translate.google.com/?sl=auto&tl=ar&text=I' +
                randomExercise.ar +
                '&op=translate'
              "
              class="btn btn-sm"
              >Audio</a
            >
          </div>
        </div>
      </div>
      <div class="card-actions justify-center mt-6 pt-2">
        <button
          class="btn btn-primary"
          @click="isRevealed = true"
          v-if="!isRevealed"
        >
          Reveal
        </button>
        <div class="" v-else>
          <div class="flex gap-2 justify-center items-center">
            <button class="btn" @click="evaluateScore(0)">Wrong</button>
            <button class="btn" @click="evaluateScore(1)">Kind Of</button>
            <button class="btn" @click="evaluateScore(2)">Correct</button>
          </div>
        </div>
      </div>
    </div>
  </div>
  <br />
</template>

<style scoped></style>
