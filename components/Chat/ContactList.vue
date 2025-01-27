<script lang="ts" setup>
import { type ChatContactVO, RoomType } from "@/composables/api/chat/contact";
import ContextMenu from "@imengyu/vue3-context-menu";
import { WSMemberStatusEnum } from "~/types/chat/WsType";

const props = defineProps<{
  dto?: ContactPageDTO
}>();
const isLoading = ref<boolean>(false);
const setting = useSettingStore();
const user = useUserStore();
const chat = useChatStore();
const isReload = ref(false);
const pageInfo = ref({
  cursor: null as null | string,
  isLast: false,
  size: 20,
});
/**
 * 加载会话列表
 */
async function loadData(dto?: ContactPageDTO) {
  if (isLoading.value || pageInfo.value.isLast)
    return;
  isLoading.value = true;
  const { data } = await getChatContactPage({
    pageSize: pageInfo.value.size,
    cursor: pageInfo.value.cursor,
  }, user.getToken);
  if (!data) {
    return;
  }
  if (data && data.list) {
    for (const item of data.list) {
      chat.contactMap[item.roomId] = item;
    }
  }
  pageInfo.value.isLast = data.isLast;
  pageInfo.value.cursor = data.cursor;
  isLoading.value = false;
  return data.list;
}
// 改变当前会话
const theContactId = computed({
  get() {
    return chat.theContact.roomId;
  },
  set(contactId: number) {
    onChangeRoom(contactId);
  },
});
// 切换房间
async function onChangeRoom(newRoomId?: number) {
  if (!newRoomId || chat.theContact.roomId === newRoomId)
    return;
  const item = chat.contactMap[newRoomId];
  if (!item)
    return;
  await chat.setContact(item); // 提前设置当前会话
  setting.isOpenContact = false;
}
chat.onChangeRoom = onChangeRoom;
// 刷新
async function reload(size: number = 20, dto?: ContactPageDTO, isAll: boolean = true, roomId?: number) {
  if (isReload.value)
    return;
  isReload.value = true;
  if (isAll) {
    chat.contactMap = {};
    pageInfo.value.cursor = null;
    pageInfo.value.isLast = false;
    pageInfo.value.size = size;
    if (setting.isMobileSize) { // 移动端
      setting.isOpenContact = true;// 打开会话列表
      setting.isOpenGroupMember = false;// 关闭群成员列表
      setting.isOpenContactSearch = true;// 打开搜索框
    }
    await loadData(dto || props.dto);
  }
  else if (roomId) { // 刷新某一房间
    refreshItem(roomId);
  }
  isReload.value = false;
}

const isLoadRoomMap: Record<number, boolean> = {};
// 刷新某一房间
async function refreshItem(roomId: number) {
  if (!roomId || isLoadRoomMap[roomId])
    return;
  isLoadRoomMap[roomId] = true;
  try {
    const item = chat.contactMap[roomId] as ChatContactVO | undefined;
    if (item?.type !== undefined && item.type !== null)
      return;
    if (item?.type === RoomType.GROUP) {
      const res = await getChatContactInfo(roomId, user.getToken, RoomType.GROUP);
      if (res)
        chat.contactMap[roomId] = res.data;
    }
  }
  catch (error) {
    console.log(error);
  }
  finally {
    delete isLoadRoomMap[roomId];
  }
}
chat.onReloadContact = reload;

// 添加群聊
const showDialog = ref(false);
const ChatNewGroupDialogRef = ref();

// 右键菜单
const colorMode = useColorMode();
function onContextMenu(e: MouseEvent, item: ChatContactVO) {
  e.preventDefault();
  const isPin = !!chat.contactMap?.[item.roomId]?.pinTime;
  const opt = {
    x: e.x,
    y: e.y,
    theme: colorMode.preference === "dark" ? "mac dark" : "wind10",
    items: [
      { // 置顶功能
        customClass: "group",
        icon: isPin ? "i-solar:pin-bold-duotone group-btn-info group-hover:i-solar:pin-outline" : "i-solar:pin-outline group-btn-info group-hover:i-solar:pin-bold-duotone",
        label: isPin ? "取消置顶" : "置顶",
        onClick: () => {
          chat.setPinContact(item.roomId, isPin ? isTrue.FALESE : isTrue.TRUE, () => {
          });
        },
      },
      {
        customClass: "group",
        icon: "i-solar:trash-bin-minimalistic-outline group-btn-danger group-hover:i-solar:trash-bin-minimalistic-bold-duotone",
        label: "不显示聊天",
        onClick: () => {
          chat.deleteContactConfirm(item.roomId, () => {
          });
        },
      },
    ] as any,
  };
  // 群聊
  if (item.type === RoomType.GROUP) {
    // 在第一个后插入
    opt.items.splice(1, 0, {
      customClass: "group",
      icon: "i-solar:user-speak-broken group-btn-warning group-hover:i-solar:user-speak-bold-duotone",
      label: "邀请好友",
      onClick: () => {
        ChatNewGroupDialogRef.value?.reload && ChatNewGroupDialogRef.value?.reload();
        if (ChatNewGroupDialogRef.value?.form) {
          ChatNewGroupDialogRef.value.form.roomId = item.roomId;
          showDialog.value = true;
        }
      },
    });
  }
  else if (item.type === RoomType.SELFT) {
    opt.items.splice(1, 0, {
      customClass: "group",
      icon: "i-solar:user-outline group-btn-info group-hover:i-solar:user-bold-duotone",
      label: "联系TA",
      onClick: () => {
        chat.setTheFriendOpt(FriendOptType.Empty);
        navigateTo("/friend");
      },
    });
  }

  ContextMenu.showContextMenu(opt);
}
const ws = useWs();
// 成员变动消息
const stopWatch = watchDebounced(() => ws.wsMsgList.memberMsg.length, watchMemberChange, {
  immediate: false,
});
async function watchMemberChange(len: number) {
  if (!len)
    return;
  // 成员变动消息
  if (ws.wsMsgList.memberMsg.length) {
    for (const p of ws.wsMsgList.memberMsg) {
      // 新加入
      if (p.changeType === WSMemberStatusEnum.JOIN) {
        getChatContactInfo(p.roomId, user.getToken, RoomType.GROUP)?.then((res) => {
          if (res) {
            const item = chat.contactMap[p.roomId];
            if (item) { // 更新
              chat.contactMap[p.roomId] = res.data;
            }
            else { // 添加
              res.data.unreadCount = 1;
              chat.contactMap[res.data.roomId] = res.data;
              // unshift();
            }
          }
        }).finally(() => {
        });
        // 更新用户列表
        if (!p.uid)
          return;
        getCommUserInfoSe(p.uid, user.getToken).then((res) => {
          if (res.code === StatusCode.SUCCESS) {
            chat.onOfflineList.push({
              userId: p.uid,
              avatar: res.data.avatar,
              nickName: res.data.nickname,
              activeStatus: 0,
              lastOptTime: res.data.lastLoginTime,
              roleType: ChatRoomRoleEnum.MEMBER,
            });
          }
        });
      }
      else if (p.changeType === WSMemberStatusEnum.LEAVE) {
        for (let i = 0; i < chat.onOfflineList.length; i++) {
          const u = chat.onOfflineList[i];
          // 下线删除用户
          if (u && u.userId === p.uid) {
            chat.onOfflineList.splice(i, 1);
            break;
          }
        }
      }
      else if (p.changeType === WSMemberStatusEnum.DEL) {
        // 删除会话
        await chat.removeContact(p.roomId);
      }
    }
    ws.wsMsgList.memberMsg.splice(0);
  }
}

// 跳转好友页面
function toFriendPage() {
  setTimeout(() => {
    navigateTo("/friend");
  }, 200);
}


reload();
onBeforeUnmount(() => {
  stopWatch?.();
});
</script>

<template>
  <div
    class="group absolute z-4 h-full w-0 flex flex-shrink-0 flex-col overflow-hidden border-0 border-0 rounded-0 bg-white sm:(relative left-0 top-0 w-1/4 pl-0) dark:bg-dark-7"
    :class="setting.isOpenContact ? (setting.showChatMenu ? 'w-full sm:w-1/4' : 'hidden') : ''"
  >
    <!-- 搜索群聊 -->
    <div
      class="h-18 flex-row-c-c flex-shrink-0 px-4 transition-200 transition-height"
      :class="setting.isMobileSize && !setting.isOpenContactSearch ? '!h-0 overflow-y-hidden' : ''"
    >
      <ElInput
        id="search-contact" v-model.lazy="chat.searchKeyWords" name="search-content" class="mr-2" type="text" clearable
        autocomplete="off" :prefix-icon="ElIconSearch" minlength="2" maxlength="30" placeholder="搜索"
      />
      <!-- 添加 -->
      <el-dropdown placement="bottom-end" popper-class="dropdown-btns" trigger="click">
        <BtnElButton plain style="width: 2rem;transition: 200ms;">
          <i i-carbon:add-large p-2 />
        </BtnElButton>
        <template #dropdown>
          <el-dropdown-menu>
            <el-dropdown-item :icon="ElIconUser" @click="toFriendPage">
              添加好友
            </el-dropdown-item>
            <!-- divided -->
            <el-dropdown-item divided :icon="ElIconPlus" @click="showDialog = true">
              新建群聊
            </el-dropdown-item>
          </el-dropdown-menu>
        </template>
      </el-dropdown>
    </div>
    <!-- 会话列表 -->
    <el-scrollbar wrap-class="w-full h-full" class="contact-list" wrapper-class="relative">
      <!-- 添加骨架屏 -->
      <div v-if="isReload" class="animate-(fade-in duration-200) overflow-y-auto">
        <ChatContactSkeleton v-for="i in 10" :key="i" />
      </div>
      <ListAutoIncre
        :immediate="true"
        :auto-stop="false"
        loading-class="op-0"
        :no-more="pageInfo.isLast"
        @load="loadData(dto)"
      >
        <ListTransitionGroup :immediate="false">
          <div
            v-for="room in chat.getContactList"
            :key="room.roomId"
            class="contact w-full text-sm"
            :class="{
              'is-pin': room.pinTime,
              'is-checked': room.roomId === theContactId,
            }"
            @contextmenu.stop="onContextMenu($event, room)"
            @click="onChangeRoom(room.roomId)"
          >
            <div
              class="flex items-center gap-3 truncate px-4 py-3 transition-200 transition-shadow sm:(w-full p-4 px-5) hover:bg-[#7c7c7c1a] text-color"
            >
              <el-badge
                :hidden="!room.unreadCount" :max="99" :value="room.unreadCount"
                class="h-2.4rem w-2.4rem flex-shrink-0"
              >
                <CardElImage
                  :src="BaseUrlImg + room.avatar" fit="cover"
                  class="h-2.4rem w-2.4rem object-cover card-default"
                />
              </el-badge>
              <div class="flex flex-1 flex-col justify-between truncate">
                <div flex truncate>
                  <p truncate font-400>
                    {{ room.name }}
                  </p>
                  <span ml-a w-fit flex-shrink-0 text-right text-10px text-mini>
                    {{ formatContactDate(room.activeTime) }}
                  </span>
                </div>
                <p mt-1 flex text-small>
                  <small
                    class="flex-1 truncate"
                    :class="{ 'text-[var(--el-color-info)] font-600': room.unreadCount }"
                  >
                    {{ room.text }}
                  </small>
                  <small v-if="room.pinTime" class="ml-a flex-shrink-0 text-[var(--el-color-primary-light-3)] dark:text-light-500">
                    置顶
                  </small>
                </p>
              </div>
            </div>
          </div>
        </ListTransitionGroup>
        <template #done>
          <div class="my-4 w-full text-center text-mini">
            {{ pageInfo.isLast ? '没有更多了' : '' }}
          </div>
        </template>
      </ListAutoIncre>
    </el-scrollbar>
    <!-- 新建群聊 -->
    <LazyChatNewGroupDialog ref="ChatNewGroupDialogRef" v-model="showDialog" />
  </div>
</template>

<style lang="scss" scoped>
.contact-list {
  .contact {
    &.is-pin {
      --at-apply: "bg-light-500 dark:bg-dark";
    }

    &.is-checked {
      background-color: #7c7c7c1a;
    }
  }
}

:deep(.el-scrollbar__bar) {
  opacity: 0.4;
}
</style>
