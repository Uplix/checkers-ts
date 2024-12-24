<script setup>
  import { reactive, ref } from 'vue';
  import Piece from './components/Piece.vue';

  //true for black turn, false for red turn
  const turn = ref(true)
  const highlightedPieces = ref([])
  const doubleJump = ref(false)

  // 'r' for red, 'b' for black, 'e' for empty
  const getBoard = ()=>{
    var theBoard = [];

    for(var i = 0; i < 8; i++){
      let row = [];
      for(var j = 0; j < 8; j++){
        let block = {
          piece: { type: "e", display: null, id: null },
          color: i % 2 === j % 2 ? "r" : "b",
        }
        if(i < 3 && ((i%2 == 0 && j%2 == 1) || (i%2 == 1 && j%2 == 0))){
          block.piece.type = "b";
          block.piece.display = Piece;
          block.piece.id = `${i} ${j}`;
        }else if(i > 4 && ((i%2 == 0 && j%2 == 1) || (i%2 == 1 && j%2 == 0))){
          block.piece.type = "r";
          block.piece.display = Piece;
          block.piece.id = `${i} ${j}`;
        }
        
        row.push(block);
      }
      theBoard.push(row);
    }
    return theBoard
  }

  const board = reactive(getBoard())

  let draggedPiece = ref(null);
  let sourcePosition = ref(null);

  function handleDragStart(event, row, column) {
    // event.preventDefault();
    if(doubleJump.value){
      draggedPiece.value = board[row][column].piece;
      sourcePosition.value = { row, column };
      if(!board[row][column].selected){
        unselect();
        turn.value = !turn.value
        doubleJump.value = false
        event.preventDefault();
      }
    }else{
      unselect()
      if((board[row][column].piece.type.toLowerCase() == "b" && turn.value) || (board[row][column].piece.type.toLowerCase() == "r" && !turn.value)){
        draggedPiece.value = board[row][column].piece;
        sourcePosition.value = { row, column };
        highlightedPieces.value.push({row:row,column:column})
        
        const locations = findValidLocations(row, column);
        board[row][column].selected = locations.length == 0?"error":"good";
        for(const location of locations){
          board[location.row][location.column].highlight = true;
          highlightedPieces.value.push({row:location.row,column:location.column})
        }
      }else{
        event.preventDefault();
      }
    }
  }

  function handleDragOver(event) {
    event.preventDefault();
  }

  function handleDrop(row, column) {
    if (!draggedPiece.value) return;

    const validLocations = findValidLocations(sourcePosition.value.row, sourcePosition.value.column);
    const isValidMove = validLocations.some((loc) => loc.row === row && loc.column === column);
    if(doubleJump.value){
      if(isValidMove){
        board[sourcePosition.value.row][sourcePosition.value.column].piece = { type: "e", display: null, id: null };
        board[row][column].piece = draggedPiece.value;
        draggedPiece.value = null;
        board[row - ((row - sourcePosition.value.row)/2)][column - ((column - sourcePosition.value.column)/2)].piece = { type: "e", display: null, id: null}
        unselect();
        turn.value = !turn.value
        doubleJump.value = false
      }else{
        unselect();
        turn.value = !turn.value
        doubleJump = !doubleJump
      }
    }else if(isValidMove) {
      board[sourcePosition.value.row][sourcePosition.value.column].piece = { type: "e", display: null, id: null };
      board[row][column].piece = draggedPiece.value;
      checkKing(row, column)

      if(Math.abs(sourcePosition.value.row - row) == 2){
        board[row - ((row - sourcePosition.value.row)/2)][column - ((column - sourcePosition.value.column)/2)].piece = { type: "e", display: null, id: null}
        unselect();

        const locations = findValidLocations(row, column)
        let doubleLocations = []
        for(const location of locations){
          if(Math.abs(location.row - row) == 2){
            doubleLocations.push(location)
          }
        }
        if(doubleLocations.length > 0){
          doubleJump.value = true
          highlightedPieces.value.push({row:row,column:column})
          board[row][column].selected = "good"
          for(const location of doubleLocations){
            board[location.row][location.column].highlight = true;
            highlightedPieces.value.push({row:location.row,column:location.column})
          }
        }else{
          turn.value = !turn.value
        }
      }else{
        turn.value = !turn.value
        unselect();
      }
    }else{
      unselect();
    }
    draggedPiece.value = null;
    sourcePosition.value = null;
  }

  function findValidLocations(row, column){
    let validLocations = []
    if(board[row][column].piece.type.toLowerCase() == "b" || board[row][column].piece.type == "R"){
      if(row + 1 < board.length){
        if(column - 1 >= 0){
          if(board[row + 1][column - 1].piece.type == "e"){
            validLocations.push({
              row: row+1,
              column: column - 1
            })
          }else if(row + 2 < board.length && column - 2 >= 0 && ((board[row][column].piece.type.toLowerCase() == "b" && board[row + 1][column - 1].piece.type.toLowerCase() == "r") || (board[row][column].piece.type.toLowerCase() == "r" && board[row+1][column-1].piece.type.toLowerCase() == "b")) && board[row + 2][column - 2].piece.type == "e"){
            validLocations.push({
              row:row+2,
              column:column-2
            })
          }
        }
        if(column + 1 < board.length){
          if(board[row+1][column + 1].piece.type == "e"){
            validLocations.push({
              row: row+1,
              column:column+1
            })
          }else if(row + 2 < board.length && column + 2 < board.length &&((board[row][column].piece.type.toLowerCase() == "r" && board[row + 1][column + 1].piece.type.toLowerCase() == "b") || (board[row][column].piece.type.toLowerCase() == "b" && board[row+1][column+1].piece.type.toLowerCase() == "r")) && board[row + 2][column + 2].piece.type == "e"){
            validLocations.push({
              row:row+2,
              column:column+2
            })
          }
        }
      }
    } 
    if(board[row][column].piece.type.toLowerCase() == "r" ||  board[row][column].piece.type == "B"){
      if(row - 1 >= 0){
        if(column - 1 >= 0){
          if(board[row - 1][column - 1].piece.type == "e"){
            validLocations.push({
              row: row-1,
              column: column - 1
            })
          }else if(row - 2 >= 0 && column - 2 >= 0 && ((board[row][column].piece.type.toLowerCase() == "r" && board[row - 1][column - 1].piece.type.toLowerCase() == "b") || (board[row][column].piece.type.toLowerCase() == "b" && board[row-1][column-1].piece.type.toLowerCase() == "r")) && board[row - 2][column - 2].piece.type == "e"){
            validLocations.push({
              row:row-2,
              column:column-2
            })
          }
        }
        if(column + 1 < board.length){
          if(board[row-1][column + 1].piece.type == "e"){
            validLocations.push({
              row: row-1,
              column:column+1
            })
          }else if(row - 2 >= 0 && column + 2 < board.length && ((board[row][column].piece.type.toLowerCase() == "r" && board[row - 1][column + 1].piece.type.toLowerCase() == "b") || (board[row][column].piece.type.toLowerCase() == "b" && board[row-1][column+1].piece.type.toLowerCase() == "r")) && board[row - 2][column + 2].piece.type == "e"){
            validLocations.push({
              row:row-2,
              column:column+2
            })
          }
        }
      }
    }
    return validLocations;
  }

  function unselect(){
    for(const pieceLocation of highlightedPieces.value){
      if('highlight' in board[pieceLocation.row][pieceLocation.column]) delete board[pieceLocation.row][pieceLocation.column].highlight
      if('selected' in board[pieceLocation.row][pieceLocation.column]) delete board[pieceLocation.row][pieceLocation.column].selected
    }
    highlightedPieces.value = [];
  }

  function clickSpot(row, column){
    if(doubleJump.value){
      if(board[row][column].highlight){
        let pieceHolder = board[highlightedPieces.value[0].row][highlightedPieces.value[0].column].piece
        board[highlightedPieces.value[0].row][highlightedPieces.value[0].column].piece = { type: "e", display: null, id: null}
        board[row][column].piece = pieceHolder
        checkKing(row, column)
        board[row - ((row - highlightedPieces.value[0].row)/2)][column - ((column - highlightedPieces.value[0].column)/2)].piece = { type: "e", display: null, id: null}
        unselect();
        turn.value = !turn.value
        doubleJump.value = false
      }else{
        unselect();
        turn.value = !turn.value
        doubleJump.value = false
      }
    }else if(board[row][column].highlight){
      let pieceHolder = board[highlightedPieces.value[0].row][highlightedPieces.value[0].column].piece
      board[highlightedPieces.value[0].row][highlightedPieces.value[0].column].piece = { type: "e", display: null, id: null}
      board[row][column].piece = pieceHolder
      checkKing(row, column)

      if(Math.abs(highlightedPieces.value[0].row - row) == 2){
        board[row - ((row - highlightedPieces.value[0].row)/2)][column - ((column - highlightedPieces.value[0].column)/2)].piece = { type: "e", display: null, id: null}
        unselect();

        const locations = findValidLocations(row, column)
        let doubleLocations = []
        for(const location of locations){
          if(Math.abs(location.row - row) == 2){
            doubleLocations.push(location)
          }
        }
        if(doubleLocations.length > 0){
          doubleJump.value = true
          highlightedPieces.value.push({row:row,column:column})
          board[row][column].selected = "good"
          for(const location of doubleLocations){
            board[location.row][location.column].highlight = true;
            highlightedPieces.value.push({row:location.row,column:location.column})
          }
        }else{
          turn.value = !turn.value
        }
      }else{
        turn.value = !turn.value
        unselect();
      }
    }else{
      unselect()
      if((board[row][column].piece.type.toLowerCase() == "b" && turn.value) || (board[row][column].piece.type.toLowerCase() == "r" && !turn.value)){
        highlightedPieces.value.push({row:row,column:column})
        
        const locations = findValidLocations(row, column);
        board[row][column].selected = locations.length == 0?"error":"good";
        for(const location of locations){
          board[location.row][location.column].highlight = true;
          highlightedPieces.value.push({row:location.row,column:location.column})
        }
      }
    }
  }

  function checkKing(row, column){
    if(row == board.length - 1 || row == 0){
      board[row][column].piece.type = board[row][column].piece.type.toUpperCase();
    }
  }
  
</script>

<template>
  <div class="w-screen h-screen flex flex-col items-center justify-center bg-[url('../src/assets/background1.jpg')]  bg-cover">
    <div v-for="(row, rIndex) in board" class="w-fit h-fit flex flex-row shadow-xl shadow-sky-400" :id="'row ' + rIndex">
      <div v-for="(spot, sIndex) in row" @click="clickSpot(rIndex, sIndex)" :id="'row ' + rIndex + 'column ' + sIndex" :class="[
        'w-20 h-20',
        spot.color === 'r' ? 'bg-red-600' : 'bg-zinc-800',
        spot.selected === 'error' ? ' border-2 border-yellow-400' : spot.selected ? 'border-2 border-blue-50 border-opacity-65' : '',
        spot.highlight?'border-2 border-blue-500':'',
        'transition-all duration-150 hover:bg-opacity-90'
      ]"
      @dragover="handleDragOver"
      @drop="handleDrop(rIndex, sIndex)"
      >
        <component v-if="spot.piece.type != 'e'" :is="spot.piece.display" :id="spot.piece.id" :pieceType="spot.piece.type" 
        draggable="true" 
        @dragstart="(e)=>handleDragStart(e, rIndex, sIndex)" 
       ></component>
      </div>
    </div>
  </div>
</template>