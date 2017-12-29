<template>
	<div class="wrap">
		<div class="vue_load" :style="{ 'transform': transform }">
			<slot></slot>
			<slot name="bottom">
				<div class="load__bottom">
					<span class="load__arrow" :class="{'rotate': Status=='drop'}" v-show="Status != 'loading'">↑</span>
		      <span class="load__bottom__loading" v-show="Status == 'loading'">
		        <div class="spinner-snake"></div>
		      </span>
	      </div>
			</slot>	
		</div>
	</div>
</template>

<script>
	export default {
		data() {
			return {
				Status: '',
				autoFill: false,
				bottomReached: false,
				Dropped: false,
				maxDistance: 0,
				direction: '',
				translate: 0,
				startScrollTop: 0,
				containerFilled: false,
				distanceIndex: 2,
				bottomMethod: () => {
					setTimeout(() => {
						console.log(1)
						this.onBottomLoaded();
					}, 1500)
				}
			}
		},

		mounted() {
			this.init();
		},

		watch: {
			Status(val) {
				this.$emit('status-change', val);
        switch (val) {
          case 'pull':
            this.bottomText = this.bottomPullText;
            break;
          case 'drop':
            this.bottomText = this.bottomDropText;
            break;
          case 'loading':
            this.bottomText = this.bottomLoadingText;
            break;
        }
      }
		},

		computed: {
      transform() {
        return this.translate === 0 ? null : 'translate3d(0, ' + this.translate + 'px, 0)';
      }
    },

    methods: {
    	init() {
        this.Status = 'pull';
        this.scrollEventTarget = this.getScrollEventTarget(this.$el);
        this.fillContainer();
        this.bindTouchEvents();
      },

      onBottomLoaded() {
        this.Status = 'pull';
        this.Dropped = false;
        this.$nextTick(() => {
          if (this.scrollEventTarget === window) {
            document.body.scrollTop += 50;
          } else {
            this.scrollEventTarget.scrollTop += 50;
          }
          this.translate = 0;
        });
        if (!this.containerFilled) {
          this.fillContainer();
        }
      },

      fillContainer() {
        if (this.autoFill) {
          this.$nextTick(() => {
            if (this.scrollEventTarget === window) {
              this.containerFilled = this.$el.getBoundingClientRect().bottom >=
                document.documentElement.getBoundingClientRect().bottom;
            } else {
              this.containerFilled = this.$el.getBoundingClientRect().bottom >=
                this.scrollEventTarget.getBoundingClientRect().bottom;
            }
            if (!this.containerFilled) {
              this.Status = 'loading';
              this.bottomMethod();
            }
          });
        }
      },

      getScrollTop(element) {
        if (element === window) {
          return Math.max(window.pageYOffset || 0, document.documentElement.scrollTop);
        } else {
          return element.scrollTop;
        }
      },

      getScrollEventTarget(element) {
        let currentNode = element;
        while (currentNode && currentNode.tagName !== 'HTML' &&
          currentNode.tagName !== 'BODY' && currentNode.nodeType === 1) {
          let overflowY = document.defaultView.getComputedStyle(currentNode).overflowY;
          if (overflowY === 'scroll' || overflowY === 'auto') {
            return currentNode;
          }
          currentNode = currentNode.parentNode;
        }
        return window;
      },

      checkBottomReached() {
        if (this.scrollEventTarget === window) {
          /**
           * fix:scrollTop===0
           */
          return document.documentElement.scrollTop || document.body.scrollTop + document.documentElement.clientHeight >= document.body.scrollHeight;
        } else {
          return this.$el.getBoundingClientRect().bottom <= this.scrollEventTarget.getBoundingClientRect().bottom + 1;
        }
      },

      bindTouchEvents() {
        this.$el.addEventListener('touchstart', this.handleTouchStart);
        this.$el.addEventListener('touchmove', this.handleTouchMove);
        this.$el.addEventListener('touchend', this.handleTouchEnd);
      },

      handleTouchStart(event) {
        this.startY = event.touches[0].clientY;
        this.startScrollTop = this.getScrollTop(this.scrollEventTarget);
        this.bottomReached = false;
        if (this.Status !== 'loading') {
          this.Status = 'pull';
          this.Dropped = false;
        }
      },

      handleTouchMove(event) {
        if (this.startY < this.$el.getBoundingClientRect().top && this.startY > this.$el.getBoundingClientRect().bottom) {
          return;
        }
        this.currentY = event.touches[0].clientY;
        let distance = (this.currentY - this.startY) / this.distanceIndex;
        this.direction = distance > 0 ? 'down' : 'up';

        if (this.direction === 'up') {
          this.bottomReached = this.bottomReached || this.checkBottomReached();
        }

        if (this.direction === 'up' && this.bottomReached && this.Status !== 'loading') {
          event.preventDefault();
          event.stopPropagation();
          if (this.maxDistance > 0) {
            this.translate = Math.abs(distance) <= this.maxDistance
              ? this.getScrollTop(this.scrollEventTarget) - this.startScrollTop + distance : this.translate;
          } else {
            this.translate = this.getScrollTop(this.scrollEventTarget) - this.startScrollTop + distance;
          }
          if (this.translate > 0) {
            this.translate = 0;
          }
          this.Status = -this.translate >= 40 ? 'drop' : 'pull';
        }
        console.log(this.Status);
        this.$emit('translate-change', this.translate);
      },

      handleTouchEnd() {
        // if (this.direction === 'down' && this.getScrollTop(this.scrollEventTarget) === 0 && this.translate > 0) {
        //   this.topDropped = true;
        //   if (this.topStatus === 'drop') {
        //     this.translate = '50';
        //     this.topStatus = 'loading';
        //     this.topMethod();
        //   } else {
        //     this.translate = '0';
        //     this.topStatus = 'pull';
        //   }
        // }
        if (this.direction === 'up' && this.bottomReached && this.translate < 0) {
          this.Dropped = true;
          this.bottomReached = false;
          if (this.Status === 'drop') {
            this.translate = '-50';
            this.Status = 'loading';
            this.bottomMethod();
          } else {
            this.translate = '0';
            this.Status = 'pull';
          }
        }
        this.$emit('translate-change', this.translate);
        this.direction = '';
      }
    }
	}
</script>

<style>
.wrap {
	overflow: scroll;
}

.vue_load {
	/*overflow: hidden;*/
}

.load__bottom {
	margin-bottom: -50px;
	height: 50px;
	text-align: center;
}

.load__bottom .load__bottom__loading {
	display: inline-block;
}

.spinner-snake {
  border: 4px solid transparent;
  border-top-color: rgb(204, 204, 204);
  border-left-color: rgb(204, 204, 204);
  border-bottom-color: rgb(204, 204, 204);
  height: 18px;
  width: 18px;
  border-radius: 50%;
  animation: rotate .8s infinite linear;
}

@keyframes rotate {
  0% {
    transform: rotate(0deg);
  }

  100% {
    transform: rotate(1turn);
  }
}

.load__arrow {
	display: inline-block;
  transition: all 0.2s linear;
}

.rotate {
  transform: rotate(180deg);
}
</style>

