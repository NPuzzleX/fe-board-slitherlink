<script lang="ts">
  import { onMount } from "svelte";
  import { loadState, getPuzzle, postAnswer } from "../helper/backend";
  import * as iconList from "../helper/iconList";

  //------------- SCRIBBLE BOARD CONTROL -------------
  let CV: HTMLCanvasElement
  let ctx: CanvasRenderingContext2D
  let mainDiv: HTMLDivElement;

  let isMouseDown: boolean = false;
  let isScrolled:boolean = false;

  let pointSize: number = 2;
  $: intervalSize = Math.floor(pointSize*1.3);

  let brushColour: string = '#000000';
  let posX: number = 0;
  let posY: number = 0;
  let writeMode: number = 2;
  /*
    0 : erase
    1 : write
    2 : move
  */
  $: cursorType = (writeMode == 0) ? "" : ((writeMode == 1) ? "" : "grab")

  let originalHeight = 1200;
  let originalWidth = 1200;
  let scaling = 1;
  let zoomFactor = 1;

  let cellFontSize = 12;
  $: {
    cellFontSize = (originalHeight > originalWidth ? (originalHeight * scaling/rowNum) : (originalWidth * scaling/colNum))/2;
  }

  let mainAreaHeight: number;
  let mainAreaWidth: number;
  let optionHeight: number;
  let fitPage:boolean = true;
  let optionHidden: boolean = false;

  $: isOverflowWidth = fitPage ? false : (!mainDiv ? false : (mainDiv.clientWidth < originalWidth*scaling));
  $: isOverflowheight = fitPage ? false : (!mainDiv ? false : (mainDiv.clientHeight < originalHeight*scaling));

  $: {
    if (fitPage) {
      if ((mainAreaWidth)/colNum > (mainAreaHeight - (optionHidden ? 0 : optionHeight))/rowNum) {
        zoomFactor = (mainAreaHeight - (optionHidden ? 0 : optionHeight))/mainAreaHeight;
        scaling = mainAreaHeight/originalHeight*zoomFactor;
      } else {
        zoomFactor = (mainAreaWidth)/mainAreaWidth;
        scaling = mainAreaWidth/originalWidth*zoomFactor;
      }
      cellFontSize = (originalHeight > originalWidth ? (originalHeight * scaling/rowNum) : (originalWidth * scaling/colNum))/2;
    } else {
      if (zoomFactor <= 0) {
        zoomFactor = 1;
      }

      if ((mainAreaWidth - 15)/colNum > (mainAreaHeight - 15)/rowNum) {
        scaling = mainAreaHeight/originalHeight*zoomFactor;
      } else {
        scaling = mainAreaWidth/originalWidth*zoomFactor;
      }
      cellFontSize = (originalHeight > originalWidth ? (originalHeight * scaling/rowNum) : (originalWidth * scaling/colNum))/2;
    }
  }

  let scribbleBoard: boolean = false;

  let isPenDown: boolean = false;
  let ongoingTouches = [];

  let originalBoard: any = undefined;

  function eraserBtn() {
    writeMode = 0;
    pointSize *= 25;
  }

  function writeBtn(bColour: string) {
    if (bColour != "") {
      brushColour = bColour;
    }
    pointSize = 2;
    writeMode = 1;
  }


  function mouseMove(e: MouseEvent) {
    if (isMouseDown) {
      e.preventDefault();
      const newX = (e.clientX - CV.getBoundingClientRect().left)/scaling;
      const newY = (e.clientY - CV.getBoundingClientRect().top)/scaling;

      if (writeMode == 0) {
        if ((posX > 0) && (posY > 0)) {
          const dotCount = Math.floor(Math.sqrt(Math.pow(newX - posX, 2) + Math.pow(newY - posY, 2))/intervalSize);
          if (dotCount > 0) {
            const xInterval = (newX - posX)/dotCount;
            const yInterval = (newY - posY)/dotCount;
            for (let i = 0; i < dotCount; i++) {
              ctx.clearRect(posX + xInterval*i - pointSize*0.5, posY + yInterval*i - pointSize*0.5, pointSize, pointSize);
            }
          }
        }

        posX = newX;
        posY = newY;
        ctx.clearRect(posX - pointSize*0.5, posY - pointSize*0.5, pointSize, pointSize);
      } else if (writeMode == 1) {
        if ((posX > 0) && (posY > 0)) {
          const dotCount = Math.floor(Math.sqrt(Math.pow(newX - posX, 2) + Math.pow(newY - posY, 2))/intervalSize);
          if (dotCount > 0) {
            const xInterval = (newX - posX)/dotCount;
            const yInterval = (newY - posY)/dotCount;
            for (let i = 0; i < dotCount; i++) {
              drawDot(posX + xInterval*i, posY + yInterval*i);
            }
          }
        }

        posX = newX;
        posY = newY;
        drawDot(posX, posY);
      } else if (writeMode == 2) {
        mainDiv.scrollTop = mainDiv.scrollTop - e.movementY;
        mainDiv.scrollLeft = mainDiv.scrollLeft - e.movementX;
      }
    }
  }

  function mouseDown(e: MouseEvent) {
    isMouseDown = true;
    posX = (e.clientX - CV.getBoundingClientRect().left)/scaling;
    posY = (e.clientY - CV.getBoundingClientRect().top)/scaling;
  }

  function mouseUp() {
    if (isMouseDown) {
      isMouseDown = false;
      saveStorageScribble();
      posX = -1;
      posY = -1;
    }
  }

  function mouseLeave () {
    if (isMouseDown) {
      isMouseDown = false;
      saveStorageScribble();
      posX = -1;
      posY = -1;
    }
    isScrolled = false;
  }

  function pointerDown(e: PointerEvent) {
    if (e.pointerType == "pen") {
      isPenDown = true;
    }
  }

  function touchStart (e: TouchEvent) {
    if ((writeMode != 2) || (isPenDown)) {
      const touches = e.changedTouches;

      for (let i = 0; i < touches.length; i++) {
        ongoingTouches.push(copyTouch(touches[i]));
      }      
    }
  }

  function touchMove(e: TouchEvent) {
    if ((writeMode != 2) || (isPenDown)) {
      e.preventDefault();
      const touches = e.changedTouches;

      for (let i = 0; i < touches.length; i++) {
        const idx = ongoingTouchIndexById(touches[i].identifier);

        if (idx >= 0) {
          const newX = (touches[i].clientX - CV.getBoundingClientRect().left)/scaling;
          const newY = (touches[i].clientY - CV.getBoundingClientRect().top)/scaling;

          if (writeMode == 0) {
            if ((ongoingTouches[idx].x > 0) && (ongoingTouches[idx].y > 0)) {
              const dotCount = Math.floor(Math.sqrt(Math.pow(newX - ongoingTouches[idx].x, 2) + Math.pow(newY - ongoingTouches[idx].y, 2))/intervalSize);
              if (dotCount > 0) {
                const xInterval = (newX - ongoingTouches[idx].x)/dotCount;
                const yInterval = (newY - ongoingTouches[idx].y)/dotCount;
                for (let i = 0; i < dotCount; i++) {
                  ctx.clearRect(ongoingTouches[idx].x + xInterval*i - pointSize*0.5, ongoingTouches[idx].y + yInterval*i - pointSize*0.5, pointSize, pointSize);
                }
              }
            }

            ctx.clearRect(ongoingTouches[idx].x - pointSize*0.5, ongoingTouches[idx].y - pointSize*0.5, pointSize, pointSize);
          } else {
            if ((ongoingTouches[idx].x > 0) && (ongoingTouches[idx].y > 0)) {
              const dotCount = Math.floor(Math.sqrt(Math.pow(newX - ongoingTouches[idx].x, 2) + Math.pow(newY - ongoingTouches[idx].y, 2))/intervalSize);
              if (dotCount > 0) {
                const xInterval = (newX - ongoingTouches[idx].x)/dotCount;
                const yInterval = (newY - ongoingTouches[idx].y)/dotCount;
                for (let i = 0; i < dotCount; i++) {
                  drawDot(ongoingTouches[idx].x + xInterval*i, ongoingTouches[idx].y + yInterval*i);
                }
              }
            }

            drawDot(ongoingTouches[idx].x, ongoingTouches[idx].y);
          }

          ongoingTouches.splice(idx, 1, copyTouch(touches[i])); 
        }
      }
    }    
  }

  function touchEnd(e: TouchEvent) {
    if ((writeMode != 2) || (isPenDown)) {
      
      ongoingTouches.splice(0, ongoingTouches.length);

      if (isPenDown) {
        isPenDown = false;
      }
    }
  }

  function touchCancel(e: TouchEvent) {
    if ((writeMode != 2) || (isPenDown)) {
      e.preventDefault();
      ongoingTouches.splice(0, ongoingTouches.length);

      if (isPenDown) {
        isPenDown = false;
      }

      saveStorageScribble();
    }
  }

  function copyTouch({ identifier, clientX, clientY }) {
    return {
      id: identifier, 
      x: (clientX - CV.getBoundingClientRect().left)/scaling, 
      y: (clientY - CV.getBoundingClientRect().top)/scaling 
    };
  }

  function ongoingTouchIndexById(idToFind) {
    for (let i = 0; i < ongoingTouches.length; i++) {
      const id = ongoingTouches[i].id;

      if (id === idToFind) {
        return i;
      }
    }
    return -1;
  }

  function drawDot(x: number, y: number) {
    ctx.fillStyle = brushColour;
    ctx.beginPath();
    ctx.arc(x - pointSize*0.5, y - pointSize*0.5, pointSize, 0, 2 * Math.PI);
    ctx.fill();
  }

  function mouseMoveMain(e: MouseEvent) {
    if (isMouseDown) {
      e.preventDefault();

      if (Math.abs(e.movementX) + Math.abs(e.movementY) > 2)  {
        mainDiv.scrollTop = mainDiv.scrollTop - e.movementY;
        mainDiv.scrollLeft = mainDiv.scrollLeft - e.movementX;
      }
    }
  }

  function mouseDownMain(e: MouseEvent) {
    isMouseDown = true;
  }

  function mouseUpMain() {
    if (isMouseDown) {
      isMouseDown = false;
    }
  }

  function mouseLeaveMain () {
    if (isMouseDown) {
      isMouseDown = false;
    }
    isScrolled = false;
  }

  function clearCanvas() {
    ctx.clearRect(0, 0, originalHeight, originalWidth);
    saveStorageScribble();
  }

  function checkTouchscreen(): boolean {
    if ("maxTouchPoints" in navigator) {
      return (navigator.maxTouchPoints > 0);
    } else if ("msMaxTouchPoints" in navigator) {
      //@ts-ignore
      return (navigator.msMaxTouchPoints > 0);
    } else {
      const mQ = matchMedia?.("(pointer:coarse)");
      if (mQ?.media === "(pointer:coarse)") {
        return (!!mQ.matches);
      } else if ("orientation" in window) {
        return (true);
      } else {
        //@ts-ignore
        const UA = navigator.userAgent;
        return (
          /\b(BlackBerry|webOS|iPhone|IEMobile)\b/i.test(UA) ||
          /\b(Android|Windows Phone|iPad|iPod)\b/i.test(UA)
        );
      }
    }
  }

  onMount(async () => {
    const target = document.getElementById("optionBoard");
   
    const observer = new MutationObserver(e => {
      if (e[0].attributeName == "style") {
        if (e[0].target.firstChild.parentElement.style.display == "none") {
          optionHidden = true;
        } else {
          optionHidden = false;
        }
      }
		})
		
	  observer.observe(target, { attributeFilter: ["style"] });

    if (!sessionStorage.getItem("PuzzleId")) {
      console.log("INVALID PUZZLE_ID");
      window.postMessage("INVALID PUZZLE_ID", window.origin);
      return;
    }

    const tempData = await getPuzzle();
    if (!tempData) {
      console.log("INVALID PUZZLE_ID");
      window.postMessage("INVALID PUZZLE_ID", window.origin);
      return;
    }

    if (tempData.type != "Slitherlink") {
      console.log("INVALID PUZZLE_ID");
      window.postMessage("INVALID PUZZLE_ID", window.origin);
      return;
    }

    try {
      ptype = tempData.type;
      originalBoard = JSON.parse(JSON.stringify(tempData.data));
      Val = tempData.data.map((e: number[]) => {
        return (e.map(e2 => {
          let a: boardData = {
            v: e2,
            w: false
          }
          return (a);
        }))
      });
    } catch (error) {
      console.log("INVALID PUZZLE_ID");
      window.postMessage("INVALID PUZZLE_ID", window.origin);
    }

    colNum = Val.length;
    rowNum = Val[0].length;

    hBar = Array.from({length: rowNum + 1}, () => {
      return new Array(colNum).fill(0);
    });
    hBarw = Array.from({length: rowNum + 1}, () => {
      return new Array(colNum).fill(false);
    });
    vBar = Array.from({length: rowNum}, () => {
      return new Array(colNum + 1).fill(0);
    });
    vBarw = Array.from({length: rowNum}, () => {
      return new Array(colNum + 1).fill(false);
    });

    let ctxCont = CV.getContext("2d");
    if (ctxCont) {
      ctx = ctxCont;
    }

    CV.height = originalHeight;
    CV.width = originalWidth;

    scribbleBoard = checkTouchscreen();
    load();
  });

  //------------- PLAY BOARD -------------
  type boardData = {
    v: number
    /*
      -1: empty
      0-3: number
    */
    w: boolean // confirmed wrong
  }
  
  let colNum: number = 5;
  let rowNum: number = 5;

  let Val: boardData[][] = [];
  let ptype: string = "";
  let hBar: number[][];
  let hBarw: boolean[][];
  let vBar: number[][];
  let vBarw: boolean[][];

  function doubleClickMouse(e: MouseEvent) {
    document.elementsFromPoint(e.clientX, e.clientY).every(e => {
      if (e.className.indexOf("bar") != -1) {
        const id = e.id.split(" ");
        
        if (id.length != 3) {
          return true;
        } else {
          barClick(parseInt(id[0]), parseInt(id[1]), (id[2] == "h") ? true : false);
          return false;
        }
      } else {
        return true;
      }
    })
  }

  function barEnter(i: number, j: number, horizontal: boolean) {
    if (horizontal) {
      if (hBar[i][j] != 1.0) {
        hBar[i][j] = 0.3;
      }
    } else {
      if (vBar[i][j] != 1.0) {
        vBar[i][j] = 0.3;
      }
    }
  }

  function barLeave(i: number, j: number, horizontal: boolean) {
    if (horizontal) {
      if (hBar[i][j] != 1.0) {
        hBar[i][j] = 0.0;
      }
    } else {
      if (vBar[i][j] != 1.0) {
        vBar[i][j] = 0.0;
      }
    }
  }

  function barClick(i: number, j: number, horizontal: boolean) {
    if (isScrolled) {
      isScrolled = false;
      return;
    }

    if (horizontal) {
      if (hBar[i][j] != 1.0) {
        hBar[i][j] = 1.0;
      } else {
        hBar[i][j] = 0.0;
      }
    } else {
      if (vBar[i][j] != 1.0) {
        vBar[i][j] = 1.0;
      } else {
        vBar[i][j] = 0.0;
      }
    }
    saveStorageState();
  }

  //------------- SAVE LOAD STATE -------------
  let processing: boolean = false;

  function load() {
    if (!processing) {
      processing = true;
      loadState().then((e) => {
        if (!e) {
          console.log("NO DATA");
          processing = false;
          saveStorageState();
          sessionStorage.setItem("TEMPSCRIBBLE", CV.toDataURL("image/webp", 0.5));
        } else {
          clearCanvas();
          var image = new Image();
          image.onload = function() {
              ctx.drawImage(image, 0, 0);
          };
          image.src = e["cvs"];

          Val = e["game"]["b"].map((e: number[]) => {
            return (e.map(e2 => {
              let a: boardData = {
                v: e2,
                w: false
              }
              return (a);
            }));
          });

          hBar = e["game"]["h"];
          vBar = e["game"]["v"];

          saveStorageState();
          sessionStorage.setItem("TEMPSCRIBBLE", e["cvs"]);

          processing = false;       
        }
      }).catch(err => {
        console.log("ERROR LOADING" + err);
        processing = false;
      })
    }
  }

  function resetState() {
    if (scribbleBoard) {
      clearCanvas();
    }
    
    Val = originalBoard.map((e: number[]) => {
      return (e.map(e2 => {
        let a: boardData = {
          v: e2,
          w: false
        }
        return (a);
      }));
    });

    for (let i = 0; i < hBar.length; i++) {
      for (let j = 0; j < hBar[i].length; j++) {
        hBar[i][j] = 0;
        hBarw[i][j] = false;
      }      
    }
    for (let i = 0; i < vBar.length; i++) {
      for (let j = 0; j < vBar[i].length; j++) {
        vBar[i][j] = 0;
        vBarw[i][j] = false;
      }      
    }

    saveStorageState();
  }

  function saveStorageState() {
    sessionStorage.setItem("TEMPSTATE", JSON.stringify({
      b : Val.map(e => {
        return (e.map(e2 => {
          return e2.v;
        }));
      }),
      v : vBar,
      h : hBar,
    }));
    sessionStorage.setItem("save", "Y");
  }

  function saveStorageScribble() {
    sessionStorage.setItem("TEMPSCRIBBLE", CV.toDataURL("image/webp", 0.5));
    sessionStorage.setItem("save", "Y");
  }

  //------------- CHECK ANSWER -------------
  let checking: boolean = false;
  function checkAnswer() {
    if (!checking) {
      checking = true;
      let clear = true;

      for (let i = 0; i < Val.length; i++) {
        for (let j = 0; j < Val[i].length; j++) {
          if (Val[i][j].v != -1) {
            let bars = 0;
            if (hBar[i][j]) {bars++};
            if (hBar[i + 1][j]) {bars++};
            if (vBar[i][j]) {bars++};
            if (vBar[i][j + 1]) {bars++};
            if (bars != Val[i][j].v) {
              clear = false;
              Val[i][j].w = true;
            }
          }
        }
      }

      for (let i = 0; i < hBar.length; i++) {
        for (let j = 0; j < hBar[i].length; j++) {
          if (hBar[i][j] > 0.9) {
            let connection = [0, 0];

            if (i < rowNum) {
              if (vBar[i][j + 1] > 0.9) {
                connection[1]++;
              }
            }
            if (j < colNum - 1) {
              if (hBar[i][j + 1] > 0.9) {
                connection[1]++;
              }
            }
            if (i > 0) {
              if (vBar[i - 1][j + 1] > 0.9) {
                connection[1]++;
              }
            }

            if (i < rowNum) {
              if (vBar[i][j] > 0.9) {
                connection[0]++;
              }
            }
            if (j > 0) {
              if (hBar[i][j - 1] > 0.9) {
                connection[0]++;
              }
            }
            if (i > 0) {
              if (vBar[i - 1][j] > 0.9) {
                connection[0]++;
              }
            }

            if ((connection[0] != 1) || (connection[1] != 1)) {
              hBarw[i][j] = true;
              clear = false;
            }
          }
        }      
      }

      for (let i = 0; i < vBar.length; i++) {
        for (let j = 0; j < vBar[i].length; j++) {
          if (vBar[i][j] > 0.9) {
            let connection = [0, 0];

            if (j < colNum) {
              if (hBar[i + 1][j] > 0.9) {
                connection[1]++;
              }
            }
            if (i < rowNum - 1) {
              if (vBar[i + 1][j] > 0.9) {
                connection[1]++;
              }
            }
            if (j > 0) {
              if (hBar[i + 1][j - 1] > 0.9) {
                connection[1]++;
              }
            }

            if (j < rowNum) {
              if (hBar[i][j] > 0.9) {
                connection[0]++;
              }
            }
            if (i > 0) {
              if (vBar[i - 1][j] > 0.9) {
                connection[0]++;
              }
            }
            if (j > 0) {
              if (hBar[i][j - 1] > 0.9) {
                connection[0]++;
              }
            }

            if ((connection[0] != 1) || (connection[1] != 1)) {
              vBarw[i][j] = true;
              clear = false;
            }
          }
        }      
      }

      if (!clear) {
        setTimeout(() => {
          checking = false;
          for (let i = 0; i < Val.length; i++) {
            for (let j = 0; j < Val.length; j++) {
              Val[i][j].w = false;
            }
          }
          
          for (let i = 0; i < hBarw.length; i++) {
            for (let j = 0; j < hBarw[i].length; j++) {
              hBarw[i][j] = false;
            }      
          }
          
          for (let i = 0; i < vBarw.length; i++) {
            for (let j = 0; j < vBarw[i].length; j++) {
              vBarw[i][j] = false;
            }      
          }          
        }, 2000);
      } else {
        postAnswer([
          Val.map(e => {
            return (e.map(e2 => {
              return e2.v;
            }));
          }),
          vBar,
          hBar
        ], ptype).then(e => {
          if (e.resStatus == 200) {
            if (e.res) {
              console.log("PUZZLE DONE!");
              window.postMessage("PUZZLE DONE!", window.origin);
            } else {
              console.log("BACKEND CHECKING ANSWER: WRONG");
              window.postMessage("BACKEND CHECKING ANSWER: WRONG", window.origin);
              checking = false;
            }
          } else {
            console.log("ERROR CHECKING ANSWER");
            window.postMessage("ERROR CHECKING ANSWER", window.origin);
            checking = false;
          }
        }).catch(err => {
          console.log("ERROR CHECKING ANSWER " + err);
          window.postMessage("ERROR CHECKING ANSWER", window.origin);
          checking = false;
        })
      }
    }
  }
</script>

<div id="mainArea" bind:clientWidth={mainAreaWidth} bind:clientHeight={mainAreaHeight}>
  <div id="optionBoard" bind:clientHeight={optionHeight}>
    <div class="optionH">
      <div class="option" style="display: { scribbleBoard ? "flex" : "none" }; padding-right: 5px; border-right: solid 3px black">
        <div class="palette">
          <button style="background-color:#000000" on:click={() => {writeBtn("#000000");}}/>
          <button style="background-color:#d00404" on:click={() => {writeBtn("#d00404");}}/>
        </div>
        <div class="palette">
          <button style="background-color:#00bcc0" on:click={() => {writeBtn("#00bcc0");}}/>
          <button style="background-color:#32cd32" on:click={() => {writeBtn("#32cd32");}}/>
        </div>
        <button disabled={writeMode == 0} on:click={eraserBtn} title="Erase"><div class="icon">{@html iconList.eraserIco}</div><p>Eraser</p></button>
        <button disabled={writeMode == 2} on:click={() => {writeMode = 2}} title="Move" ><div class="icon">{@html iconList.arrowMoveIco}</div><p>Move</p></button>
        <button on:click={clearCanvas} title="Clear"><div class="icon newIcon">{@html iconList.newIco}</div><p>Clear</p></button>
    </div>
      <div class="option">
        <div class="option" style="position: relative; top: -0.2em;">
          <input type=checkbox bind:checked={scribbleBoard}>
          <div style="flex-direction: column; display: flex; align-items: center">
            <div class="icon boardIcon" style="margin-bottom: 10px;">{@html iconList.boardIco}</div>
            <p>Scribble Board</p>
          </div>            
        </div>
        <button on:click={resetState} title="Reset"><div class="icon">{@html iconList.arrowRotateIco}</div><p>Reset</p></button>
        <button on:click={checkAnswer} title="Check Answer"><div class="icon">{@html iconList.checkSolidIco}</div><p>Check</p></button>
      </div>
      <div class="option" style="padding-left: 5px; border-left: solid 3px black">
        <button on:click={() => {zoomFactor >= 2 ? zoomFactor-- : zoomFactor /= 2; fitPage = false;}} title="Zoom out"><div class="icon">{@html iconList.magMinIco}</div><p>Zoom Out</p></button>
        <input type=number on:input={() => {fitPage = false;}} style="width: 4em;" bind:value={zoomFactor} min="0">
        <button on:click={() => {zoomFactor >= 1 ? zoomFactor++ : zoomFactor *= 2; fitPage = false;}} title="Zoom in"><div class="icon">{@html iconList.magPlusIco}</div><p>Zoom In</p></button>
        <button on:click={() => {fitPage = true;}} title="Fit to page"><div class="icon">{@html iconList.maxIco}</div><p>Fit Page</p></button>
        <button on:click={() => {window.postMessage("TRIGGER FULLSCREEN", window.origin);}} title="Full Screen"><div class="icon">{@html iconList.enlargeIco}</div><p>Full Screen</p></button>    </div>
      </div>
    {#if scribbleBoard}
      <div class="option">
        <h3>double click to put or remove line</h3>
      </div>        
    {/if}  
  </div>  
  <div id="boardContainer" bind:this={mainDiv}
    on:scroll={() => {isScrolled = true;}}
    style="display: {isOverflowWidth ? "unset" : "flex"}; padding-left: { (isOverflowWidth ? 20 : 0)}px; padding-top: { (isOverflowheight ? 20 : 0)}px; overflow: {fitPage ? "hidden" : "scroll"};"
  >
    <div 
      id="boardArea" style="height: {originalHeight*scaling + (isOverflowheight ? 20 : 0)}px; width: {originalWidth*scaling + (isOverflowWidth ? 20 : 0)}px;"
    >
      <canvas 
          style="display: { scribbleBoard ? "initial" : "none" }; height: {originalHeight*scaling}px; width: {originalWidth*scaling}px; z-index: 4; opacity: 1.0; cursor: { cursorType };"
          bind:this={CV}
          on:mousemove={mouseMove}
          on:mouseup={mouseUp}
          on:mousedown={mouseDown}
          on:mouseleave={mouseLeave}
          on:touchstart={touchStart}
          on:touchmove={touchMove}
          on:touchend={touchEnd}
          on:touchcancel={touchCancel}
          on:pointerdown={pointerDown}
          on:dblclick={doubleClickMouse}
      ></canvas>           
      <div id="mainBoard"
        style="z-index: 3; opacity: 1.0; 
          grid-template-columns: repeat({colNum}, 1fr);
          grid-template-rows: repeat({rowNum}, 1fr);
          height: {originalHeight*scaling}px;
          width: {originalWidth*scaling}px;
          font-size: {cellFontSize}px;
          padding-left: calc({8/colNum}% + 5px);
          padding-right: calc({8/colNum}% + 5px);
          padding-top: calc({8/rowNum}% + 5px);
          padding-bottom: calc({8/rowNum}% + 5px); 
          "
          on:mousemove={mouseMoveMain}
          on:mouseup={mouseUpMain}
          on:mousedown={mouseDownMain}
          on:mouseleave={mouseLeaveMain}
      >
        {#each Val as box, i}
          {#each box as e, j}
            <div
              class="boardCell empty" class:wrong={e.w}
            >
              <div class="front-mask">
                { (e.v >= 0) ? (e.v) : "" }
              </div>
              {#if i == 0}
                <div class="boardCell dot tr"></div>
                <div id={i.toString() + " " + j.toString() + " h"}
                  class="boardCell bar t"
                  style="opacity: {hBar[i][j]};"
                  on:pointerenter={() => {barEnter(i, j, true);}}
                  on:pointerleave={() => {barLeave(i, j, true);}}
                  on:pointerdown={() => {barClick(i, j, true);}}
                  >
                  <div class="boardCell bar h" class:wrong={hBarw[i][j]}></div>
                </div>
              {/if}
              {#if j == 0}
                <div class="boardCell dot btl"></div>
                <div id={i.toString() + " " + j.toString() + " v"}
                  class="boardCell bar l"
                  style="opacity: {vBar[i][j]};"
                  on:pointerenter={() => {barEnter(i, j, false);}}
                  on:pointerleave={() => {barLeave(i, j, false);}}
                  on:pointerdown={() => {barClick(i, j, false);}}
                  >
                  <div class="boardCell bar v" class:wrong={vBarw[i][j]}></div>
                </div>
              {/if}
              {#if (i == 0) && (j == 0)}
                <div class="boardCell dot tl"></div>
              {/if}
              <div class="boardCell dot btr"></div>
              <div id={i.toString() + " " + (j + 1).toString() + " v"}
                class="boardCell bar r"
                style="opacity: {vBar[i][j + 1]};"
                on:pointerenter={() => {barEnter(i, j + 1, false);}}
                on:pointerleave={() => {barLeave(i, j + 1, false);}}
                on:pointerdown={() => {barClick(i, j + 1, false);}}
                >
                <div class="boardCell bar v" class:wrong={vBarw[i][j + 1]}></div>
              </div>
              <div id={(i + 1).toString() + " " + j.toString() + " h"}
                class="boardCell bar b"
                style="opacity: {hBar[i + 1][j]};"
                on:pointerenter={() => {barEnter(i + 1, j, true);}}
                on:pointerleave={() => {barLeave(i + 1, j, true);}}
                on:pointerdown={() => {barClick(i + 1, j, true);}}
                >
                <div class="boardCell bar h" class:wrong={hBarw[i + 1][j]}></div>
              </div>
            </div>
          {/each}
        {/each}
      </div>
    </div>
  </div>
</div>

<style>
  #mainArea {
    display: flex;
    flex-direction: column;
    height: 100%;
    width: 100%;
  }

  #boardContainer {
    justify-content: center;
  }

  /*   SCRIBBLE BOARD   */
  canvas {
    position: absolute;
    top: 0px;
    left: 0px;
  }

  /*   MAIN PANEL   */
  #mainBoard {
    position: absolute;
    top: 0px;
    left: 0px;
    display: grid;
    border-width: 5px;
    border-style: solid;
  }

#boardArea {
  border-radius: 10px;
  position: relative;
  overflow: visible;
}

.boardCell {
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: inherit;
  position: relative;
}

.front-mask {
  color: inherit;
  background-color: inherit;
  pointer-events: none;
}

.dot {
  position: absolute;
  width: 16%;
  height: 16%;
  border-width: 0px;
  border-radius: 50%;
  z-index: 2;
}

.dot.tl {
  top: calc(-8% - 1px);
  left: calc(-8% - 1px);
}

.dot.tr {
  top: calc(-8% - 1px);
  right: calc(-8% - 1px);
}

.dot.btl {
  bottom: calc(-8% - 1px);
  left: calc(-8% - 1px);
}

.dot.btr {
  bottom: calc(-8% - 1px);
  right: calc(-8% - 1px);
}

.bar {
  position: absolute;
  z-index: 1;
}

.bar.v {
  width: 3px;
  height: 100%;
}

.bar.h {
  width: 100%;
  height: 3px;
}

.bar.t {
  width: 100%;
  height: 40%;
  left: 0%;
  top: calc(-20% - 1px);
  background-color: transparent;
}

.bar.b {
  width: 100%;
  height: 40%;
  left: 0%;
  bottom: calc(-20% - 1px);
  background-color: transparent;
}

.bar.l {
  width: 40%;
  height: 100%;
  left: calc(-20% - 1px);
  top: 0%;
  background-color: transparent;
}

.bar.r {
  width: 40%;
  height: 100%;
  right: calc(-20% - 1px);
  top: 0%;
  background-color: transparent;
}

/*   MISC  */
#optionBoard {
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.optionH {
  display: flex;
  column-gap: 5px;
  align-items: baseline;
  justify-content: center;
}

.palette {
    display: flex;
    flex-direction: column;
    justify-content: space-evenly;
    height: 100%;
  }

#miniOptionBoard {
  display: flex;
  flex-direction: row-reverse;
  padding-right: 15px;
}

.option {
  display: flex;
  column-gap: 5px;
  align-items: baseline;
  justify-content: center;
}

.option button[disabled] {
  cursor: not-allowed;
}

.option button {
  padding: 0.5em;
}

/* SVG CONTROL */
button {
  display: flex;
  flex-direction: column;
  align-items: center;
  row-gap: 2px;
}
</style>