<script setup>
  import { reactive, ref } from 'vue';
  import Piece from './components/Piece.vue';

  //true for black turn, false for red turn
  const turn = ref(true)
  const highlightedPieces = ref([])
  const doubleJump = ref(false)

  // 'r' for red, 'b' for black, 'e' for empty
  // 'R' for red king, 'B' for black king
  
  // initializes the board
  const getBoard = ()=>{
    var theBoard = [];

    for(var i = 0; i < 8; i++){
      let row = [];
      for(var j = 0; j < 8; j++){
        let block = {
          piece: { type: "e", display: null, id: null },
          color: i % 2 === j % 2 ? "r" : "b",
        }
        // adds the black pieces
        if(i < 3 && ((i%2 == 0 && j%2 == 1) || (i%2 == 1 && j%2 == 0))){
          block.piece.type = "b";
          block.piece.display = Piece;
          block.piece.id = `${i} ${j}`;
        }
        // adds the red pieces
        else if(i > 4 && ((i%2 == 0 && j%2 == 1) || (i%2 == 1 && j%2 == 0))){
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

  // initializes piece dragging
  function handleDragStart(event, row, column) {
    // if a double jump possibility exists, check if the double jump piece is being dragged. (if it's not, cancel and switch to next turn)
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
      // make sure the correct piece is being dragged for the turn
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

  // handles the dropping of the piece
  function handleDrop(row, column) {
    if (!draggedPiece.value) return;

    const validLocations = findValidLocations(sourcePosition.value.row, sourcePosition.value.column);
    const isValidMove = validLocations.some((loc) => loc.row === row && loc.column === column);
    // if a double jump possibility exists, check if the piece is being dropped at the possible double jump location
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
    }
    // if the piece is being dropped at a valid location (not a double jump instance)
    else if(isValidMove) {
      board[sourcePosition.value.row][sourcePosition.value.column].piece = { type: "e", display: null, id: null };
      board[row][column].piece = draggedPiece.value;
      checkKing(row, column)

      // check if a piece was skipped in the turn and now needs to be removed
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
        // check if a double jump is possible (if it is keep the turn and activate double jump, if not switch turns)
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

  // finds the valid locations for a piece to move to (including double jump possibilities)
  function findValidLocations(row, column){
    let validLocations = []
    // this is for pieces moving down the board
    if(board[row][column].piece.type.toLowerCase() == "b" || board[row][column].piece.type == "R"){
      if(row + 1 < board.length){
        // down and left
        if(column - 1 >= 0){
          if(board[row + 1][column - 1].piece.type == "e"){
            validLocations.push({
              row: row+1,
              column: column - 1
            })
            // check if a piece can be skipped down and left
          }else if(row + 2 < board.length && column - 2 >= 0 && ((board[row][column].piece.type.toLowerCase() == "b" && board[row + 1][column - 1].piece.type.toLowerCase() == "r") || (board[row][column].piece.type.toLowerCase() == "r" && board[row+1][column-1].piece.type.toLowerCase() == "b")) && board[row + 2][column - 2].piece.type == "e"){
            validLocations.push({
              row:row+2,
              column:column-2
            })
          }
        }
        // down and right
        if(column + 1 < board.length){
          if(board[row+1][column + 1].piece.type == "e"){
            validLocations.push({
              row: row+1,
              column:column+1
            })
            // check if a piece can be skipped down and right
          }else if(row + 2 < board.length && column + 2 < board.length &&((board[row][column].piece.type.toLowerCase() == "r" && board[row + 1][column + 1].piece.type.toLowerCase() == "b") || (board[row][column].piece.type.toLowerCase() == "b" && board[row+1][column+1].piece.type.toLowerCase() == "r")) && board[row + 2][column + 2].piece.type == "e"){
            validLocations.push({
              row:row+2,
              column:column+2
            })
          }
        }
      }
    } 
    // this is for pieces moving up the board
    if(board[row][column].piece.type.toLowerCase() == "r" ||  board[row][column].piece.type == "B"){
      if(row - 1 >= 0){
        // up and left
        if(column - 1 >= 0){
          if(board[row - 1][column - 1].piece.type == "e"){
            validLocations.push({
              row: row-1,
              column: column - 1
            })
            // check if a piece can be skipped up and left
          }else if(row - 2 >= 0 && column - 2 >= 0 && ((board[row][column].piece.type.toLowerCase() == "r" && board[row - 1][column - 1].piece.type.toLowerCase() == "b") || (board[row][column].piece.type.toLowerCase() == "b" && board[row-1][column-1].piece.type.toLowerCase() == "r")) && board[row - 2][column - 2].piece.type == "e"){
            validLocations.push({
              row:row-2,
              column:column-2
            })
          }
        }
        // up and right
        if(column + 1 < board.length){
          if(board[row-1][column + 1].piece.type == "e"){
            validLocations.push({
              row: row-1,
              column:column+1
            })
            // check if a piece can be skipped up and right
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

  // unselects all highlighted pieces (highlighted pieces are tracked through the highlightedPieces array)
  function unselect(){
    for(const pieceLocation of highlightedPieces.value){
      if('highlight' in board[pieceLocation.row][pieceLocation.column]) delete board[pieceLocation.row][pieceLocation.column].highlight
      if('selected' in board[pieceLocation.row][pieceLocation.column]) delete board[pieceLocation.row][pieceLocation.column].selected
    }
    highlightedPieces.value = [];
  }

  // handles the clicking of a spot on the board
  function clickSpot(row, column){
    // if a double jump possibility exists, check if the possible double jump location is being clicked
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
    }
    // if a double jump possibility does not exist and a possible move is being clicked
    else if(board[row][column].highlight){
      let pieceHolder = board[highlightedPieces.value[0].row][highlightedPieces.value[0].column].piece
      board[highlightedPieces.value[0].row][highlightedPieces.value[0].column].piece = { type: "e", display: null, id: null}
      board[row][column].piece = pieceHolder
      checkKing(row, column)
      // check if a piece was skipped in the turn and now needs to be removed
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
        // check if a double jump is possible (if it is keep the turn and activate double jump, if not switch turns)
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
    }
    // if a non possible move is being clicked, check if the spot being clicked has a piece that matches the turn
    else{
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

  // checks if a piece has reached the end of the board and needs to be kinged
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