<template>
  <fragment>
    <PagePreloader v-if="!contentHeight" />
    <main class="chat w-full overflow-hidden">
      <section ref="header" class="chat__header bg-senary">
        <base-container
          class="chat__header__container flex justify-between items-center"
        >
          <div class="flex justify-between items-center">
            <figure class="avatar mx-3">
              <img :src="companion.avatar" alt="companion-avatar" />
            </figure>
            <div class="chat__header__companion grid">
              <span class="chat__header__companion__name text-secondary">{{
                companion.name
              }}</span>
              <small class="chat__header__companion__action text-tertiary">{{
                companion_action
              }}</small>
            </div>
          </div>
        </base-container>
      </section>
      <section
        ref="content"
        :style="{
          height: `${contentHeight}px`,
        }"
        class="chat__content overflow-auto"
      >
        <base-container v-show="messages">
          <div v-for="(message, key) in messages" :key="message.id">
            <span
              v-if="dateCondition(key)"
              class="chat__content__date text-tertiary block text-center"
            >
              {{ $moment(message.created_at).format("dddd") }}
            </span>
            <div
              class="chat__content__msg flex"
              :class="{
                'justify-start': message.type == 0,
                'justify-end': message.type == 1,
              }"
            >
              <div class="w-max">
                <div
                  v-if="message.type == 1"
                  class="chat__content__msg__container chat__content__msg__container--mine bg-quaternary"
                >
                  <p
                    v-if="message.text"
                    class="chat__content__msg__container__text text-secondary"
                  >
                    {{ message.text }}
                  </p>
                </div>
                <div
                  v-else
                  class="chat__content__msg__container chat__content__msg__container--companion bg-senary"
                >
                  <p
                    v-if="message.text"
                    class="chat__content__msg__container__text text-secondary"
                  >
                    {{ message.text }}
                  </p>
                </div>
                <div
                  class="flex mt-1.5"
                  :class="{ 'justify-end': message.type == 1 }"
                >
                  <span class="chat__content__msg__date text-tertiary">
                    {{ $moment(message.created_at).format("LT") }}
                  </span>
                </div>
              </div>
              <figure
                v-if="message.type == 1"
                class="chat__content__msg__avatar avatar h-max grid self-end pb-6 ml-4"
              >
                <img :src="message.user.avatar" alt="mine-avatar" />
              </figure>
            </div>
          </div>
        </base-container>
      </section>
      <section ref="footer" class="chat__footer">
        <base-container class="chat__footer__container flex justify-between">
          <div class="relative w-full">
            <input
              ref="messageBar"
              type="text"
              v-model="message"
              placeholder="Введите сообщение..."
              class="chat__footer__container__field overflow-y-auto w-full break-all bg-senary text-secondary"
            />
            <button
              class="chat__footer__container__field__send cursor-pointer absolute top-3 right-4"
              @click="sendMessage"
            >
              <BaseImage logo="send" />
            </button>
          </div>
        </base-container>
      </section>
    </main>
  </fragment>
</template>

<script>
import { mapGetters } from "vuex";
import socket from "~/plugins/socket.io.js";
import uniqid from "uniqid";
import PagePreloader from "../components/PagePreloader.vue";
export default {
  layout: "chat",
  data() {
    return {
      messages: null,
      message: "",
      contentHeight: null,
      isTyping: false,
    };
  },
  computed: {
    ...mapGetters("chat", ["companion", "user"]),
    companion_action() {
      return this.isTyping ? "Typing..." : "Online";
    },
  },
  watch: {
    messages() {
      this.scrollTo();
    },
    message(val) {
      if (val.length > 0) {
        this.isTyping = true;
        return;
      }
      this.isTyping = false;
    },
  },
  methods: {
    getContentHeight() {
      if (
        this.$refs.header &&
        this.$refs.footer &&
        this.$refs.header.clientHeight &&
        this.$refs.footer.clientHeight
      ) {
        this.contentHeight = ` ${
          document.body.clientHeight -
          this.$refs.header.clientHeight -
          this.$refs.footer.clientHeight
        }`;
      }
      this.scrollTo();
    },
    scrollTo() {
      if (this.$refs.content) {
        this.$refs.content.scrollTop = this.scrollHeight;
      }
    },
    dateCondition(key) {
      return (
        this.$moment(this.messages[key].created_at).format("dddd") !=
        (this.messages[key - 1]
          ? this.$moment(this.messages[key - 1].created_at).format("dddd")
          : null)
      );
    },
    async sendMessage() {
      if (this.message.length > 0) {
        const message = {
          id: uniqid(),
          type: 1,
          text: this.message,
          created_at: new Date(),
          user: this.user,
          files: this.files,
        };
        await socket.emit("send-message", message);
      }
      this.message = "";
    },
  },
  mounted() {
    socket.on("load_dialog", (messages) => {
      this.messages = messages;
    });
    socket.on("new-message", (message, replied) => {
      this.messages.push(message, replied);
      this.$nextTick(() => {
        this.scrollTo();
      });
    });
    this.getContentHeight();
    window.addEventListener("resize", this.getContentHeight);
    this.$nextTick(() => {
      this.scrollTo();
    });
    this.$refs.messageBar.addEventListener("keydown", (event) => {
      if ((event.keyCode == 10 || event.keyCode == 13) && event.ctrlKey) {
        this.sendMessage();
      }
    });
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.getContentHeight);
  },
  components: { PagePreloader },
};
</script>

<style lang="scss">
.avatar {
  img {
    height: 49px;
    width: 49px;
    object-fit: cover;
    border-radius: 100%;
  }
  &--hover {
    img {
      height: 100px;
      width: 100px;
      object-fit: cover;
    }
  }
}
.chat__header__arrow {
  path {
    fill: $secondary;
  }
}
</style>

<style lang="scss" scoped>
%date {
  font-family: "Panton-SemiBold";
  font-size: 12px;
  letter-spacing: 0.05em;
}
.chat {
  &__header {
    &__container {
      padding: 1rem 1.5rem;
    }
    &__arrow {
      height: 28px;
      width: 28px;
      path {
        fill: $tertiary;
      }
    }
    &__companion {
      &__name {
        font-family: "Panton-Bold";
        font-size: 18px;
      }
      &__action {
        font-family: "Panton-SemiBold";
        font-size: 14.5px;
      }
    }
  }
  &__content {
    &__date {
      @extend %date;
      line-height: 14px;
      &:not(:first-child) {
        margin-top: 1rem;
      }
    }
    &__msg {
      margin-top: 1rem;
      &__date {
        @extend %date;
        line-height: 18px;
        margin-right: 10px;
      }
      &__container {
        max-width: 50vw;
        width: max-content;
        font-family: "Panton-Regular";
        font-size: 14px;
        line-height: 21px;
        letter-spacing: 0.05em;
        border-radius: 10px;
        padding-top: 10px;
        padding-bottom: 10px;
        &--mine {
          padding-right: 25px;
          padding-left: 15px;
          position: relative;
          &::after {
            content: "";
            display: block;
            position: absolute;
            right: -8px;
            bottom: 10px;
            border-top: 10px solid transparent;
            border-bottom: 10px solid transparent;
            border-right: 10px solid $quaternary;
            transform: rotate(180deg);
            border-top-right-radius: 2px;
            border-bottom-right-radius: 2px;
          }
        }
        &--companion {
          padding-right: 15px;
          padding-left: 25px;
        }
      }
    }
  }
  &__footer {
    min-height: 74px;
    height: max-content;
    &__container {
      &__field {
        border-radius: 7px;
        outline: none;
        padding: 12px calc(15px + 30px + 5px) 12px 15px;
        font-family: "Panton-SemiBold";
        font-size: 14px;
        line-height: 20px;
        letter-spacing: 0.05em;
        &:empty {
          &:before {
            content: attr(data-text);
            color: $tertiary;
            cursor: text;
          }
        }
        &__attach {
          border-radius: 7px;
          padding: 8.5px 25px;
        }
      }
    }
  }
}
</style>
