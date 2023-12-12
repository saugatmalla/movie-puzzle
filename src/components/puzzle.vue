<template>
    <div class="puzzle-container">
        <div class="stats-container">
            <span>Lives: {{ game.lives }}</span>
            <span>Score: {{ game.score }}</span>
        </div>
        <div v-if="isLoading">
            Please wait while we are generating a puzzle for you...
        </div>
        <div v-else>
            <form class="user-input" @submit.prevent="checkAnswer">
                <div class="question">
                    {{ puzzle['0'] }}
                </div>

                <div>
                    <label for="input">Movie Name</label>
                    <input ref="userInputField" type="text" name="input" id="input" v-model="userInput" />
                </div>

                <button ref="submitButton">Check</button>

                <div class="assistantResponse">
                    {{ assistantResponse }}
                </div>
            </form>

            <div class="hint-container">
                <button ref="showHintButton" @click="showHint">Show Hint</button>
                <ul>
                    <li v-for="hint in userHints">{{ hint }}</li>
                </ul>
            </div>

        </div>
    </div>
</template>

<script setup>
import OpenAI from 'openai'
import { ref, onMounted } from 'vue';

const MOVIE_JSON = "https://raw.githubusercontent.com/theapache64/top250/master/top250.json"

// OpenAI constants
const ASSISTANT_ID = "asst_ApE9ewNSXZJKe8UZuvXbTL1l"
const RETRIEVAL_INTERVAL = 7000

// instantiate OpenAI
const openai = new OpenAI({ apiKey: import.meta.env.VITE_OPEN_AI_API_KEY, dangerouslyAllowBrowser: true })

const puzzle = ref({})
const isLoading = ref(true)
const thread = ref({})
const run = ref({})
const messages = ref({})
const assistantResponse = ref('')
const userInput = ref('')

const userHints = ref([])
const game = {
    lives: 3,
    score: 100
}

// dom refs
const submitButton = ref(null)
const userInputField = ref(null)
const showHintButton = ref(null)

const showHint = () => {
    game.score -= 10 * (userHints.value.length + 1)
    userHints.value.push(puzzle.value[userHints.value.length + 1])
    if (userHints.value.length === 3) {
        showHintButton.value.disabled = true
    }
}

const checkAnswer = async () => {
    if (userInput.value.trim().length === 0) {
        return
    }

    // disable submit button
    submitButton.value.disabled = true

    const prompt = `The player's response is in triple asterisks here ***${userInput.value}***. If the player doesn't answer about the game and talk about something else, provide a funny response and add that they have lost one life. If they ask for clues or hints, direct the player to click the show hint button and add that their lack of knowledge has cost them points. Provide the response in json format in 2 line key-value pair that starts from 0 - 1, first line should be Boolean value, true for correct answer, and second line should be a funny response (don't use exact movie title in response). `
    await openai.beta.threads.messages.create(
        thread.value.id,
        { role: "user", content: prompt },
    );

    runAssistant()

    checkMessages(() => {
        const json = messages.value.data[0].content[0].text.value
        // remove first and last line
        const jsonObject = JSON.parse(json.split('\n').slice(1, -1).join('\n'))

        if (jsonObject['0'] === true) {
            userInputField.value.disabled = true
            console.log('correct answer');
        } else {
            if (game.lives === 0) {
                userInputField.value.disabled = true
                game.score = 0
                assistantResponse.value = jsonObject['1']
                console.log('game over');
                return
            }
            submitButton.value.disabled = false
            userInput.value = ''
            game.lives -= 1
            game.score -= 10
            console.log('wrong answer, lives -- ', game.lives);
        }
        assistantResponse.value = jsonObject['1']
    })
}

const checkMessages = (fn) => {
    const interval = setInterval(async () => {
        const runRetrieve = await openai.beta.threads.runs.retrieve(
            thread.value.id,
            run.value.id
        );

        if (runRetrieve.status === 'completed') {
            clearInterval(interval)
            messages.value = await openai.beta.threads.messages.list(thread.value.id)
            fn()
        }
    }, RETRIEVAL_INTERVAL)
}

const runAssistant = async () => {
    // run assistant
    run.value = await openai.beta.threads.runs.create(
        thread.value.id,
        {
            assistant_id: ASSISTANT_ID,
        }
    );
}

const generatePuzzle = async (prompt) => {
    // create new thread
    thread.value = await openai.beta.threads.create();

    // add message to thread
    await openai.beta.threads.messages.create(
        thread.value.id,
        { role: "user", content: prompt },
    );

    runAssistant()

    checkMessages(async () => {
        // first message is the puzzle
        const json = messages.value.data[0].content[0].text.value
        // remove first and last line
        const jsonObject = JSON.parse(json.split('\n').slice(1, -1).join('\n'))

        puzzle.value = jsonObject

        // hide loading
        isLoading.value = false
    })
}

const init = async () => {

    // select movie from IMDB json
    const movies = await (await fetch(MOVIE_JSON)).json()
    const movie = movies[Math.floor(Math.random() * 100)]

    if (!movie) {
        throw new Error("Failed to get movie")
    }

    // get movie title
    const movieTitle = movie.name
    console.log('movie title -- ', movieTitle);

    // prompt to create puzzle from movie title
    // const prompt = `Create a puzzle from the movie title ${movieTitle}. First give a general puzzle question in a modern poetic style but use simple words, then provide three clues to guess the movie title. 1 easy clue and 2 hard clues. Easy clue: Hard clue 1: Hard clue 2: make sure the response is json string exatly 4 lines. the key for general question and each clue should be 0 to 3. I will play the game in the following message prompts. If I answer the movie correctly or incorrectly, provide response in json format in 2 line key-value pair that starts from 0 - 1, first line should be Boolean value, and second line should be a funny response. If I don't answer about the game and talk about something else, again provide response in json format in 2 line key-value pair that starts from 0 - 1, first line should be Boolean value, and second line should be a funny response and add that they have lost one life.`

    const prompt = `Create a puzzle from the movie title ${movieTitle}. First give a general puzzle question in a modern poetic style but use simple words, then provide three clues to guess the movie title. 1 easy clue, 1 lead actor clue and 1 hard clue. Make sure the response is json string exatly 4 lines. the key for general question and each clue should be 0 to 3. I will play the game in the following message prompts.`

    // spin OpenAI
    await generatePuzzle(prompt)
}

onMounted(() => {
    init()
})

</script>

<style scoped>
.puzzle-container {
    width: min(90vw, 500px);
    margin-top: 80px;
}

form {
    display: flex;
    flex-direction: column;

}

label {
    display: block;
    font-weight: bold;
    margin-top: 22px;
    margin-bottom: 10px;
}

input {
    height: 40px;
    width: 100%;
    margin-bottom: 10px;
    font-size: 18px;
}

.stats-container {
    position: fixed;
    top: 20px;
    right: 20px;
    display: flex;
    justify-content: space-between;
    gap: 20px;
    margin-bottom: 24px;
}

.assistantResponse {
    margin-top: 24px;
    font-style: italic;
}

.hint-container {
    margin-top: 24px;
}

.hint-container ul {
    list-style: none;
    padding: 0;
}

.hint-container li {
    margin-bottom: 10px;
}
</style>
