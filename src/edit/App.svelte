<script lang="ts">
  import { onMount } from "svelte";
  import { postPuzzle, getPuzzle, validateToken } from "../helper/backend";
  import html2canvas from "html2canvas"
  import * as iconList from "../helper/iconList";

  let mainDiv: HTMLDivElement;
  let hidden: boolean = false;

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
  
  $: {
    if (rowNum < 1) {
      rowNum = 1;
    } else if (Math.round(rowNum) != rowNum) {
      rowNum = Math.round(rowNum);
    }
  }

  $: {
    if (colNum < 1) {
      colNum = 1;
    } else if (Math.round(colNum) != colNum) {
      colNum = Math.round(colNum);
    }
  }

  let isMouseDown:boolean = false;
  let isScrolled:boolean = false;

  function mouseMove(e: MouseEvent) {
    if (isMouseDown) {
      e.preventDefault();

      if (Math.abs(e.movementX) + Math.abs(e.movementY) > 2)  {
        mainDiv.scrollTop = mainDiv.scrollTop - e.movementY;
        mainDiv.scrollLeft = mainDiv.scrollLeft - e.movementX;
      }
    }
  }

  function mouseDown(e: MouseEvent) {
    isMouseDown = true;
  }

  function mouseUp() {
    if (isMouseDown) {
      isMouseDown = false;
    }
  }

  function mouseLeave () {
    if (isMouseDown) {
      isMouseDown = false;
    }
    isScrolled = false;
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
      return;
    }

    const tempData = await getPuzzle();
    if (!tempData) {
      console.log("INVALID PUZZLE_ID");
      window.postMessage("INVALID PUZZLE_ID", window.origin);
    }

    try {
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

  while (Val.length < rowNum) {
    let a: boardData[] = [];
    while (a.length < colNum) {
      a.push({v: -1, w: false});
    }
    Val.push(a);
  }

  $: {
    while (Val.length > rowNum) {
      Val.pop();
    }

    while (Val.length < rowNum) {
      let a: boardData[] = [];
      while (a.length < colNum) {
        a.push({v: -1, w: false});
      }
      Val.push(a);
    }

    Val.forEach(e => {
      while (e.length > colNum) {
        e.pop();
      }

      while (e.length < colNum) {
        e.push({v: -1, w: false});
      }
    })

    originalWidth = originalHeight*colNum/rowNum;
    forceRefresh();
  }

  function forceRefresh() {
    Val = Val;
  }

  function mainPanelRightClick(i: number, j: number) {
    if (isScrolled) {
      isScrolled = false;
      return;
    }

    if (Val[i][j].v == 3) {
      Val[i][j].v = -1;
    } else  {
      Val[i][j].v++;
    }
  }

  function mainPanelLeftClick(i: number, j: number) {
    if (isScrolled) {
      isScrolled = false;
      return;
    }

    if (Val[i][j].v == -1) {
      Val[i][j].v = 3;
    } else  {
      Val[i][j].v--;
    }
  }

  function clearState() {
    for(let i = 0; i < Val.length; i++) {
      for(let j = 0; j < Val[i].length; j++) {
        Val[i][j].v = -1;
        Val[i][j].w = false;
      }
    }
  }
  //------------- SAVING + THUMBNAIL GENERATOR -------------
  let thumbnailTarget: HTMLDivElement;
  const maxThumbnailSize: number = 400;
  let checking: boolean = false;
  let printMode: boolean = false;

  async function check() {
    if (!checking) {
      checking = true;
      let clear = true;
      
      for (let i = 0; i < Val.length; i++) {
        for (let j = 0; j < Val[i].length; j++) {
          if (Val[i][j].v > 0) {
            let blocked = 0;
            let hCorner = false;
            let vCorner = false;
            let zeros = [false, false, false, false];

            if (i == 0) {
              vCorner = true;
            } else if (Val[i - 1][j].v == 0) {
              zeros[0] = true;
              blocked++;
            }

            if (j == 0) {
              hCorner = true;
            } else if (Val[i][j - 1].v == 0) {
              zeros[1] = true;
              blocked++;
            }

            if (i == Val.length - 1) {
              vCorner = true;
            } else if (Val[i + 1][j].v == 0) {
              zeros[2] = true;
              blocked++;
            }

            if (j == Val[i].length - 1) {
              hCorner = true;
            } else if (Val[i][j + 1].v == 0) {
              zeros[3] = true;
              blocked++;
            }
            
            if (hCorner && vCorner && (zeros[0] || zeros[1] || zeros[2] || zeros[3])) {
              blocked++;
            }

            if (hCorner && (zeros[0] || zeros[2])) {blocked++;}
            if (vCorner && (zeros[1] || zeros[3])) {blocked++;}

            if (4 - blocked < Val[i][j].v) {
              clear = false;
              Val[i][j].w = true;
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
        }, 2000);
      } else {
        printMode = true;
        await new Promise(resolve => setTimeout(resolve, 1000));
        
        let renderedWidth = maxThumbnailSize;
        let renderedHeight = renderedWidth*originalHeight/originalWidth;

        if (renderedHeight > maxThumbnailSize) {
          renderedHeight = maxThumbnailSize;
          renderedWidth = renderedHeight*originalWidth/originalHeight;
        }

        let cvs = await html2canvas(thumbnailTarget, {
          scale: renderedWidth/thumbnailTarget.clientWidth
        });
        if (await postPuzzle({
          game: Val.map(e => {
            return e.map(e2 => {
              return e2.v;
            });
          }),
          thumbnail: cvs.toDataURL("image/webp", 0.5),
          },
          hidden)) {
          console.log("UPLAODED NEW PUZZLE");
          window.postMessage("UPLOADED NEW PUZZLE", window.origin);
        }
      }
    }
  }
</script>

<div id="mainArea" bind:clientWidth={mainAreaWidth} bind:clientHeight={mainAreaHeight}>
  <div id="optionBoard" bind:clientHeight={optionHeight}>
    <div class="optionH">
      <div class="option">
        <div class="icon">{@html iconList.colIco}</div>
        <input type=number style="width: 4em;" bind:value={colNum} min="1">
        <div class="icon">{@html iconList.rowIco}</div>
        <input type=number style="width: 4em;" bind:value={rowNum} min="1">
        <input type=checkbox bind:checked={hidden}>
        <div class="icon hiddenIcon">{@html iconList.hiddenIco}</div>Hidden
        <button on:click={clearState} title="Clear"><div class="icon newIcon">{@html iconList.newIco}</div><p>Clear</p></button>
        <button on:click={check} title="Save"><div class="icon">{@html iconList.saveIco}</div><p>Upload</p></button>
      </div>
      <div class="option" style="padding-left: 5px; border-left: solid 3px black">
        <button on:click={() => {zoomFactor >= 2 ? zoomFactor-- : zoomFactor /= 2; fitPage = false;}} title="Zoom out"><div class="icon">{@html iconList.magMinIco}</div><p>Zoom Out</p></button>
        <input type=number on:input={() => {fitPage = false;}} style="width: 4em;" bind:value={zoomFactor} min="0">
        <button on:click={() => {zoomFactor >= 1 ? zoomFactor++ : zoomFactor *= 2; fitPage = false;}} title="Zoom in"><div class="icon">{@html iconList.magPlusIco}</div><p>Zoom In</p></button>
        <button on:click={() => {fitPage = true;}} title="Fit to page"><div class="icon">{@html iconList.maxIco}</div><p>Fit Page</p></button>
        <button on:click={() => {window.postMessage("TRIGGER FULLSCREEN", window.origin);}} title="Full Screen"><div class="icon">{@html iconList.enlargeIco}</div><p>Full Screen</p></button>
      </div>
    </div>
  </div>    
  <div id="boardContainer" bind:this={mainDiv}
    on:scroll={() => {isScrolled = true;}}
    style="display: {isOverflowWidth ? "unset" : "flex"}; padding-left: { (isOverflowWidth ? 20 : 0)}px; padding-top: { (isOverflowheight ? 20 : 0)}px; overflow: {fitPage ? "hidden" : "scroll"};"
  >
    <div id="boardArea" bind:this={thumbnailTarget} style="height: {originalHeight*scaling + (isOverflowheight ? 20 : 0)}px; width: {originalWidth*scaling + (isOverflowWidth ? 20 : 0)}px;">
      <div id="mainBoard"
        style="z-index: 1; opacity: 1.0; grid-template-columns: repeat({colNum}, 1fr); grid-template-rows: repeat({rowNum}, 1fr); height: {originalHeight*scaling}px; width: {originalWidth*scaling}px; font-size: {cellFontSize}px;"
        on:mousemove={mouseMove}
        on:mouseup={mouseUp}
        on:mousedown={mouseDown}
        on:mouseleave={mouseLeave}
      >
      {#each Val as box, i}
        {#each box as e, j}
          <div id={i.toString()} 
            class="boardCell empty" class:wrong={e.w} class:printing={printMode}
            on:contextmenu|preventDefault={() => mainPanelLeftClick(i, j)} 
            on:click={() => mainPanelRightClick(i, j)}
          >
            <div class="front-mask">
              { (e.v >= 0) ? (e.v) : "" }
            </div>
            {#if i == 0}
              <div class="boardCell dot tr"></div>
            {/if}
            {#if j == 0}
              <div class="boardCell dot bl"></div>
            {/if}
            {#if (i == 0) && (j == 0)}
              <div class="boardCell dot tl"></div>
            {/if}
            <div class="boardCell dot br"></div>
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

#mainBoard {
    position: absolute;
    top: 0px;
    left: 0px;
    display: grid;
    border-width: 1px;
    border-style: dashed;
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
  border-width: 1px;
  border-style: dashed;
}

.boardCell.printing {
  border-style: none;
  border-width: 0px;
}

.front-mask {
  color: inherit;
  background-color: inherit;
}

.dot {
  position: absolute;
  width: 16%;
  height: 16%;
  border-width: 0px;
  border-radius: 50%;
  z-index: 1;
}

.dot.tl {
  top: calc(-8% - 1px);
  left: calc(-8% - 1px);
}

.dot.tr {
  top: calc(-8% - 1px);
  right: calc(-8% - 1px);
}

.dot.bl {
  bottom: calc(-8% - 1px);
  left: calc(-8% - 1px);
}

.dot.br {
  bottom: calc(-8% - 1px);
  right: calc(-8% - 1px);
}

/*   MISC  */
#optionBoard {
  display: flex;
  flex-direction: column;
  justify-content: center;
  position: sticky;
}

.optionH {
  display: flex;
  column-gap: 5px;
  align-items: baseline;
  justify-content: center;
}

.option {
  display: flex;
  column-gap: 5px;
  align-items: baseline;
  justify-content: center;
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