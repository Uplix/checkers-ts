<script setup>
    // import {defineProps} from 'vue'

    defineProps({
        pieceType:String
    })
    
    function tester(){
        alert(pieceType)
    }
    function getHref(theType){
        switch(theType){
            case "r":
                return "../src/assets/redReg.png"
            case "R":
                return "../src/assets/redKing.png"
            case "b":
                return "../src/assets/blackReg.png"
            case "B":
                return "../src/assets/blackKing.png"
        }
        return "../assets/vue.svg"
    }
    function preventer(event){
        event.preventDefault();
    }
</script>

<template>
    <div class="w-fit h-fit">
        <img :src="getHref(pieceType)" alt="piece" class="object-contain" />
    </div>
</template>