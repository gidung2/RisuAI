<script lang="ts">
    import { DatabaseIcon, DicesIcon, LanguagesIcon, MenuIcon, RefreshCcwIcon, Send } from "lucide-svelte";
    import { selectedCharID } from "../../ts/stores";
    import Chat from "./Chat.svelte";
    import { DataBase, appVer, type Message } from "../../ts/database";
    import { getCharImage } from "../../ts/characters";
    import { doingChat, sendChat } from "../../ts/process/index";
    import { findCharacterbyId, messageForm } from "../../ts/util";
    import { language } from "../../lang";
    import { translate } from "../../ts/translator/translator";
    import { alertError } from "../../ts/alert";
    import sendSound from '../../etc/send.mp3'
    import {cloneDeep} from 'lodash'
  import { processScript } from "src/ts/process/scripts";

    let messageInput = ''
    let openMenu = false
    export let openChatList = false
    let loadPages = 30
    let autoMode = false
    let rerolls:Message[][] = []
    let rerollid = -1
    let lastCharId = -1

    async function send() {
        let selectedChar = $selectedCharID
        console.log('send')
        if($doingChat){
            return
        }
        if(lastCharId !== $selectedCharID){
            rerolls = []
            rerollid = -1
        }

        let cha = $DataBase.characters[selectedChar].chats[$DataBase.characters[selectedChar].chatPage].message

        if(messageInput === ''){
            if($DataBase.characters[selectedChar].type !== 'group'){
                if(cha.length === 0 || cha[cha.length - 1].role !== 'user'){
                    if($DataBase.useSayNothing){
                        cha.push({
                            role: 'user',
                            data: '*says nothing*'
                        })
                    }
                }
            }
        }
        else{
            const char = $DataBase.characters[selectedChar]
            if(char.type === 'character'){
                cha.push({
                    role: 'user',
                    data: processScript(char,messageInput,'editinput')
                })
            }
            else{
                cha.push({
                    role: 'user',
                    data: messageInput
                })
            }
        }
        messageInput = ''
        $DataBase.characters[selectedChar].chats[$DataBase.characters[selectedChar].chatPage].message = cha
        rerolls = []
        await sendChatMain()

    }

    async function reroll() {
        if($doingChat){
            return
        }
        if(lastCharId !== $selectedCharID){
            rerolls = []
            rerollid = -1
        }
        if(rerollid < rerolls.length - 1){
            if(Array.isArray(rerolls[rerollid + 1])){
                let db = $DataBase
                rerollid += 1
                db.characters[$selectedCharID].chats[$DataBase.characters[$selectedCharID].chatPage].message = cloneDeep(rerolls[rerollid])
                $DataBase = db
            }
            return
        }
        if(rerolls.length === 0){
            rerolls.push($DataBase.characters[$selectedCharID].chats[$DataBase.characters[$selectedCharID].chatPage].message)
            rerollid = rerolls.length - 1
        }
        let cha = $DataBase.characters[$selectedCharID].chats[$DataBase.characters[$selectedCharID].chatPage].message
        if(cha.length === 0 ){
            return
        }
        openMenu = false
        const saying = cha[cha.length - 1].saying
        let sayingQu = 2
        while(cha[cha.length - 1].role !== 'user'){
            if(cha[cha.length - 1].saying === saying){
                sayingQu -= 1
                if(sayingQu === 0){
                    break
                }   
            }
            cha.pop()
        }
        $DataBase.characters[$selectedCharID].chats[$DataBase.characters[$selectedCharID].chatPage].message = cha
        await sendChatMain()
    }

    async function unReroll() {
        if(rerollid <= 0){
            return
        }
        if(lastCharId !== $selectedCharID){
            rerolls = []
            rerollid = -1
        }
        if($doingChat){
            return
        }
        if(Array.isArray(rerolls[rerollid - 1])){
            let db = $DataBase
            rerollid -= 1
            db.characters[$selectedCharID].chats[$DataBase.characters[$selectedCharID].chatPage].message = cloneDeep(rerolls[rerollid])
            $DataBase = db
        }
    }

    async function sendChatMain(saveReroll = false) {
        messageInput = ''
        try {
            await sendChat()            
        } catch (error) {
            alertError(`${error}`)
        }
        rerolls.push(cloneDeep($DataBase.characters[$selectedCharID].chats[$DataBase.characters[$selectedCharID].chatPage].message))
        rerollid = rerolls.length - 1
        lastCharId = $selectedCharID
        $doingChat = false
        if($DataBase.playMessage){
            const audio = new Audio(sendSound);
            audio.play();
        }
    }

    async function runAutoMode() {
        if(autoMode){
            autoMode = false
            return
        }
        const selectedChar = $selectedCharID
        autoMode = true
        while(autoMode){
            await sendChatMain()
            if(selectedChar !== $selectedCharID){
                autoMode = false
            }
        }
    }

    export let customStyle = ''
    let inputHeight = "44px"
    let inputEle:HTMLTextAreaElement

    async function updateInputSize() {
        if(inputEle){
            inputEle.style.height = "0";
            inputHeight = (inputEle.scrollHeight) + "px";
            inputEle.style.height = inputHeight
        }
    }

    $: updateInputSize()

</script>
<!-- svelte-ignore a11y-click-events-have-key-events -->
<div class="w-full h-full" style={customStyle} on:click={() => {
    openMenu = false
}}>
    {#if $selectedCharID < 0}
        <div class="h-full w-full flex flex-col overflow-y-auto items-center">
            <h2 class="text-4xl text-white mb-0 mt-6 font-black">RisuAI</h2>
            <h3 class="text-gray-500 mt-1">Version {appVer}</h3>
        </div>
    {:else}
        <div class="h-full w-full flex flex-col-reverse overflow-y-auto relative"  on:scroll={(e) => {
            //@ts-ignore
            const scrolled = (e.target.scrollHeight - e.target.clientHeight + e.target.scrollTop)
            if(scrolled < 100 && $DataBase.characters[$selectedCharID].chats[$DataBase.characters[$selectedCharID].chatPage].message.length > loadPages){
                loadPages += 30
            }
        }}>
            <div class="flex items-end mt-2 mb-2">
                <textarea class="text-neutral-200 p-2 bg-transparent input-text text-xl flex-grow ml-4 mr-2 border-gray-700 resize-none focus:bg-selected maxw overflow-y-hidden overflow-x-hidden"
                    bind:value={messageInput}
                    bind:this={inputEle}
                    on:keydown={(e) => {
                        if(e.key.toLocaleLowerCase() === "enter" && (!e.shiftKey)){
                            send()
                            e.preventDefault()
                        }
                        if(e.key.toLocaleLowerCase() === "m" && (e.ctrlKey)){
                            reroll()
                            e.preventDefault()
                        }
                    }}
                    on:input={updateInputSize}
                    style:height={inputHeight}
                />

                
                {#if $doingChat}
                    <div
                        class="mr-2 bg-selected flex justify-center items-center text-white w-12 h-12 rounded-md hover:bg-green-500 transition-colors">
                        <div class="loadmove" class:autoload={autoMode}>
                        </div>
                    </div>
                {:else}
                    <div on:click={send}
                        class="mr-2 bg-gray-500 flex justify-center items-center text-white w-12 h-12 rounded-md hover:bg-green-500 transition-colors"><Send />
                    </div>
                {/if}
                <div on:click={(e) => {
                    openMenu = !openMenu
                    e.stopPropagation()
                }}
                class="mr-2 bg-gray-500 flex justify-center items-center text-white w-12 h-12 rounded-md hover:bg-green-500 transition-colors"><MenuIcon />
            </div>
            </div>
            {#each messageForm($DataBase.characters[$selectedCharID].chats[$DataBase.characters[$selectedCharID].chatPage].message, loadPages) as chat, i}
                {#if chat.role === 'char'}
                    {#if $DataBase.characters[$selectedCharID].type !== 'group'}
                        {#await getCharImage($DataBase.characters[$selectedCharID].image, 'css')}
                            <Chat
                                idx={chat.index}
                                name={$DataBase.characters[$selectedCharID].name} 
                                message={chat.data}
                                img={''}
                                rerollIcon={i === 0}
                                onReroll={reroll}
                                unReroll={unReroll}
                            />
                        {:then im} 
                            <Chat
                                idx={chat.index}
                                name={$DataBase.characters[$selectedCharID].name} 
                                message={chat.data}
                                img={im}
                                rerollIcon={i === 0}
                                onReroll={reroll}
                                unReroll={unReroll}

                            />
                        {/await}
                    {:else}
                        {#await getCharImage(findCharacterbyId(chat.saying).image, 'css')}
                            <Chat
                                idx={chat.index}
                                name={findCharacterbyId(chat.saying).name} 
                                message={chat.data}
                                rerollIcon={i === 0}
                                onReroll={reroll}
                                unReroll={unReroll}
                                img={''}
                            />
                        {:then im} 
                            <Chat
                                idx={chat.index}
                                name={findCharacterbyId(chat.saying).name} 
                                rerollIcon={i === 0}
                                message={chat.data}
                                onReroll={reroll}
                                unReroll={unReroll}
                                img={im}
                            />
                        {/await}
                    {/if}
                {:else}
                    {#await getCharImage($DataBase.userIcon, 'css')}
                        <Chat
                            idx={chat.index}
                            name={$DataBase.username} 
                            message={chat.data}
                            img={''}
                        />
                    {:then im}
                        <Chat
                            idx={chat.index}
                            name={$DataBase.username} 
                            message={chat.data}
                            img={im}
                        />
                    {/await}
                {/if}
            {/each}
            {#if $DataBase.characters[$selectedCharID].chats[$DataBase.characters[$selectedCharID].chatPage].message.length <= loadPages}
                {#if $DataBase.characters[$selectedCharID].type !== 'group'}
                    {#await getCharImage($DataBase.characters[$selectedCharID].image, 'css')}
                        <Chat
                            name={$DataBase.characters[$selectedCharID].name}
                            message={ $DataBase.characters[$selectedCharID].firstMessage}
                            img={''}
                            idx={-1}
                        />      
                    {:then im}
                        <Chat
                            name={$DataBase.characters[$selectedCharID].name}
                            message={ $DataBase.characters[$selectedCharID].firstMessage}
                            img={im}
                            idx={-1}
                        />
                    {/await}
                {/if}
            {/if}
            {#if openMenu}
                <div class="absolute right-2 bottom-16 p-5 bg-darkbg flex flex-col gap-3 text-gray-200" on:click={(e) => {
                    e.stopPropagation()
                }}>
                    {#if $DataBase.characters[$selectedCharID].type === 'group'}
                        <div class="flex items-center cursor-pointer hover:text-green-500 transition-colors" on:click={runAutoMode}>
                            <DicesIcon />
                            <span class="ml-2">{language.autoMode}</span>
                        </div>
                    {/if}

                    <div class="flex items-center cursor-pointer hover:text-green-500 transition-colors" on:click={() => {
                        openChatList = true
                        openMenu = false
                    }}>
                        <DatabaseIcon />
                        <span class="ml-2">{language.chatList}</span>
                    </div>
                    {#if $DataBase.translator !== ''}
                        <div class="flex items-center cursor-pointer hover:text-green-500 transition-colors" on:click={async () => {
                            $doingChat = true
                            messageInput = (await translate(messageInput, true))
                            $doingChat = false
                        }}>
                            <LanguagesIcon />
                            <span class="ml-2">{language.translateInput}</span>
                        </div>
                    {/if}
                    <div class="flex items-center cursor-pointer hover:text-green-500 transition-colors" on:click={reroll}>
                        <RefreshCcwIcon />
                        <span class="ml-2">{language.reroll}</span>
                    </div>
                </div>

            {/if}
        </div>

    {/if}
</div>
<style>
    .maxw{
        max-width: calc(100vw - 10rem);
    }
    .loadmove {
        animation: spin 1s linear infinite;
        border-radius: 50%;
        border: 0.4rem solid rgba(0,0,0,0);
        width: 1rem;
        height: 1rem;
        border-top: 0.4rem solid #6272a4;
        border-left: 0.4rem solid #6272a4;
    }

    .autoload{
        border-top: 0.4rem solid #10b981;
        border-left: 0.4rem solid #10b981;
    }

    @keyframes spin {
        
        0% { transform: rotate(0deg); }
        100% { transform: rotate(360deg); }
    }
</style>