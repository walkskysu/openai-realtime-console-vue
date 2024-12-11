<script setup lang="ts">
import { ref, reactive, computed, watch, onMounted, onUnmounted, nextTick } from 'vue'
/**
 * Running a local relay server will allow you to hide your API key
 * and run custom logic on the server
 *
 * Set the local relay server address to:
 * REACT_APP_LOCAL_RELAY_SERVER_URL=http://localhost:8081
 *
 * This will also require you to set OPENAI_API_KEY= in a `.env` file
 * You can run it with `npm run relay`, in parallel with `npm start`
 */
const LOCAL_RELAY_SERVER_URL: string =
  import.meta.env.VITE_APP_LOCAL_RELAY_SERVER_URL || '';


import { RealtimeClient } from '@openai/realtime-api-beta';
import { ItemType } from '@openai/realtime-api-beta/dist/lib/client.js';
import { WavRecorder, WavStreamPlayer } from '../lib/wavtools/index.js';
import { instructions } from '../utils/conversation_config.js';
import { WavRenderer } from '../utils/wav_renderer';

import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome';
import Button from '../components/button/Button.vue';
import Toggle from '../components/toggle/Toggle.vue';

import Map from '../components/Map.vue';

/**
 * Type for result from get_weather() function call
 */
interface Coordinates {
  lat: number;
  lng: number;
  location?: string;
  temperature?: {
    value: number;
    units: string;
  };
  wind_speed?: {
    value: number;
    units: string;
  };
}

/**
 * Type for all event logs
 */
interface RealtimeEvent {
  time: string;
  source: 'client' | 'server';
  count?: number;
  event: { [key: string]: any };
}


/**
 * Ask user for API Key
 * If we're using the local relay server, we don't need this
 */
const apiKey = LOCAL_RELAY_SERVER_URL
  ? ''
  : localStorage.getItem('tmp::voice_api_key') ||
  prompt('OpenAI API Key') ||
  '';
if (apiKey !== '') {
  localStorage.setItem('tmp::voice_api_key', apiKey);
}

/**
 * Instantiate:
 * - WavRecorder (speech input)
 * - WavStreamPlayer (speech output)
 * - RealtimeClient (API client)
 */
const wavRecorderRef = ref<WavRecorder>(
  new WavRecorder({ sampleRate: 24000 })
);

const wavStreamPlayerRef = ref<WavStreamPlayer>(
  new WavStreamPlayer({ sampleRate: 24000 })
);
const clientRef = ref<RealtimeClient>(
  new RealtimeClient(
    LOCAL_RELAY_SERVER_URL
      ? { url: LOCAL_RELAY_SERVER_URL }
      : {
        apiKey: apiKey,
        dangerouslyAllowAPIKeyInBrowser: true,
      }
  )
);

/**
 * References for
 * - Rendering audio visualization (canvas)
 * - Autoscrolling event logs
 * - Timing delta for event log displays
 */
const clientCanvasRef = ref<HTMLCanvasElement | null>(null);
const serverCanvasRef = ref<HTMLCanvasElement | null>(null);
const eventsScrollHeightRef = ref(0);
const eventsScrollRef = ref<HTMLDivElement | null>(null);
const startTimeRef = ref<string>(new Date().toISOString());

/**
 * 所有用于显示应用程序状态的变量
 * - items: 所有对话项（dialog）
 * - realtimeEvents: 事件日志，可以展开
 * - memoryKv: set_memory() 函数的存储
 * - coords, marker: get_weather() 函数使用的位置信息
 */

// 定义 items，类似于 React 的 useState
const items = ref<ItemType[]>([]);

// 定义 realtimeEvents，事件日志
const realtimeEvents = ref<RealtimeEvent[]>([]);

// 定义 expandedEvents，存储展开状态
const expandedEvents = ref<Record<string, boolean>>({}); // Tracks expanded event IDs

// 定义 isConnected，表示连接状态
const isConnected = ref(false);

// 定义 canPushToTalk，是否可以按住说话
const canPushToTalk = ref(true);

// 定义 isRecording，是否正在录音
const isRecording = ref(false);

// 定义 memoryKv，存储 key-value 对
const memoryKv = reactive<{ [key: string]: any }>({});

// 定义 coords，存储坐标信息
const coords = ref<Coordinates | null>({
  lat: 37.775593,
  lng: -122.418137,
});

// 定义 marker，存储标记信息
const marker = ref<Coordinates | null>(null);
const isLoaded = ref(false);


const formatTime = (timestamp: string): string => {
  // 模拟 startTimeRef 类似 React 的 ref
  const t0 = new Date(startTimeRef.value).valueOf();
  const t1 = new Date(timestamp).valueOf();
  const delta = t1 - t0;

  const hs = Math.floor(delta / 10) % 100;
  const s = Math.floor(delta / 1000) % 60;
  const m = Math.floor(delta / 60_000) % 60;

  const pad = (n: number): string => {
    let s = n + '';
    while (s.length < 2) {
      s = '0' + s;
    }
    return s;
  };

  return `${pad(m)}:${pad(s)}.${pad(hs)}`;
};

let formattedMemoryKv = computed(() => {
  return JSON.stringify(memoryKv, null, 2);
})


// Helper methods
const setMemoryKv = (updateFn: (kv: Record<string, string>) => Record<string, string>) => {
  memoryKv.value = updateFn(memoryKv.value);
};

const setMarker = (newMarker: any) => {
  marker.value = { ...marker.value, ...newMarker };
};

const setCoords = (newCoords: any) => {
  coords.value = { ...coords.value, ...newCoords };
};

const setItems = (newItems: any[]) => {
  items.value = newItems;
};

const setRealtimeEvents = (updateFn: (events: any[]) => any[]) => {
  realtimeEvents.value = updateFn(realtimeEvents.value);
};

/**
 * Switch between Manual <> VAD mode for communication
 */
const changeTurnEndType = async (value: string) => {
  const client = clientRef.value;
  const wavRecorder = wavRecorderRef.value;

  if (value === 'none' && wavRecorder.getStatus() === 'recording') {
    await wavRecorder.pause();
  }
  client.updateSession({
    turn_detection: value === 'none' ? null : { type: 'server_vad' },
  });
  if (value === 'server_vad' && client.isConnected()) {
    await wavRecorder.record((data) => client.appendInputAudio(data.mono));
  }
  canPushToTalk.value = (value === 'none');
};
const onToggleChange = (isEnabled: boolean, value: string) => {
  changeTurnEndType(value);
}

/**
 * Auto-scroll the event logs
 */
watch(
  realtimeEvents,
  () => {
    const eventsEl = eventsScrollRef.value;

    if (eventsEl) {
      const scrollHeight = eventsEl.scrollHeight;

      // Only scroll if height has just changed
      if (scrollHeight !== eventsScrollHeightRef.value) {
        eventsEl.scrollTop = scrollHeight;
        eventsScrollHeightRef.value = scrollHeight;
      }
    }
  },
  { immediate: true } // Trigger the watcher immediately after component mounts
);

/**
  * Set up render loops for the visualization canvas
  */
watch(
  items,
  () => {
    const conversationEls = Array.from(
      document.body.querySelectorAll('[data-conversation-content]')
    );

    for (const el of conversationEls) {
      const conversationEl = el as HTMLDivElement;
      conversationEl.scrollTop = conversationEl.scrollHeight;
    }
  },
  { immediate: true } // Ensures the watcher runs on component mount
);

const toggleEventDetails = (eventId: string) => {
  if (expandedEvents.value[eventId]) {
    delete expandedEvents.value[eventId];
  } else {
    expandedEvents.value[eventId] = true;
  }
};


const render = async () => {
  if (isLoaded) {
    await nextTick();
    const clientCanvas = clientCanvasRef.value;
    const serverCanvas = serverCanvasRef.value;

    let clientCtx: CanvasRenderingContext2D | null = null;
    let serverCtx: CanvasRenderingContext2D | null = null;

    if (clientCanvas) {
      if (!clientCanvas.width || !clientCanvas.height) {
        clientCanvas.width = clientCanvas.offsetWidth;
        clientCanvas.height = clientCanvas.offsetHeight;
      }

      clientCtx = clientCtx || clientCanvas.getContext('2d');
      if (clientCtx) {
        clientCtx.clearRect(0, 0, clientCanvas.width, clientCanvas.height);
        const result = wavRecorderRef.value.recording
          ? wavRecorderRef.value.getFrequencies('voice')
          : { values: new Float32Array([0]) };
        WavRenderer.drawBars(
          clientCanvas,
          clientCtx,
          result.values,
          '#0099ff',
          10,
          0,
          8
        );
      }
    }

    if (serverCanvas) {
      if (!serverCanvas.width || !serverCanvas.height) {
        serverCanvas.width = serverCanvas.offsetWidth;
        serverCanvas.height = serverCanvas.offsetHeight;
      }
      serverCtx = serverCtx || serverCanvas.getContext('2d');
      if (serverCtx) {
        serverCtx.clearRect(0, 0, serverCanvas.width, serverCanvas.height);
        const result = wavStreamPlayerRef.value.analyser
          ? wavStreamPlayerRef.value.getFrequencies('voice')
          : { values: new Float32Array([0]) };
        WavRenderer.drawBars(
          serverCanvas,
          serverCtx,
          result.values,
          '#009900',
          10,
          0,
          8
        );
      }
    }

    window.requestAnimationFrame(render);
  }
};

onMounted(() => {
  const client = clientRef.value;
  const wavStreamPlayer = wavStreamPlayerRef.value
  // Set instructions and transcription
  client.updateSession({ instructions: instructions.value });
  client.updateSession({ input_audio_transcription: { model: 'whisper-1' } });

  // Add tools
  client.addTool(
    {
      name: 'set_memory',
      description: 'Saves important data about the user into memory.',
      parameters: {
        type: 'object',
        properties: {
          key: {
            type: 'string',
            description:
              'The key of the memory value. Always use lowercase and underscores, no other characters.',
          },
          value: {
            type: 'string',
            description: 'Value can be anything represented as a string',
          },
        },
        required: ['key', 'value'],
      },
    },
    async ({ key, value }: { [key: string]: any }) => {
      setMemoryKv((current) => ({ ...current, [key]: value }));
      return { ok: true };
    }
  );

  client.addTool(
    {
      name: 'get_weather',
      description:
        'Retrieves the weather for a given lat, lng coordinate pair. Specify a label for the location.',
      parameters: {
        type: 'object',
        properties: {
          lat: {
            type: 'number',
            description: 'Latitude',
          },
          lng: {
            type: 'number',
            description: 'Longitude',
          },
          location: {
            type: 'string',
            description: 'Name of the location',
          },
        },
        required: ['lat', 'lng', 'location'],
      },
    },
    async ({ lat, lng, location }: { [key: string]: any }) => {
      setMarker({ lat, lng, location });
      setCoords({ lat, lng, location });
      const response = await fetch(
        `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lng}&current=temperature_2m,wind_speed_10m`
      );
      const json = await response.json();
      const temperature = {
        value: json.current.temperature_2m,
        units: json.current_units.temperature_2m,
      };
      const wind_speed = {
        value: json.current.wind_speed_10m,
        units: json.current_units.wind_speed_10m,
      };
      setMarker({ lat, lng, location, temperature, wind_speed });
      return json;
    }
  );

  // Event listeners
  client.on('realtime.event', (realtimeEvent: any) => {
    setRealtimeEvents((current) => {
      const lastEvent = current[current.length - 1];
      if (lastEvent?.event.type === realtimeEvent.event.type) {
        lastEvent.count = (lastEvent.count || 0) + 1;
        return current.slice(0, -1).concat(lastEvent);
      } else {
        return current.concat(realtimeEvent);
      }
    });
  });

  client.on('error', (event: any) => console.error(event));

  client.on('conversation.interrupted', async () => {
    const trackSampleOffset = await wavStreamPlayer.interrupt();
    if (trackSampleOffset?.trackId) {
      const { trackId, offset } = trackSampleOffset;
      await client.cancelResponse(trackId, offset);
    }
  });

  client.on('conversation.updated', async ({ item, delta }: any) => {
    const updatedItems = client.conversation.getItems();
    if (delta?.audio) {
      wavStreamPlayer.add16BitPCM(delta.audio, item.id);
    }
    if (item.status === 'completed' && item.formatted.audio?.length) {
      const wavFile = await WavRecorder.decode(item.formatted.audio, 24000, 24000);
      item.formatted.file = wavFile;
    }
    setItems(updatedItems);
  });

  setItems(client.conversation.getItems());
  isLoaded.value = true; // Start the render loop
  render();
});

onUnmounted(() => {
  isLoaded.value = false; // Stop the render loop
});

const connectConversation = async () => {
  const client = clientRef.value;
  const wavRecorder = wavRecorderRef.value;
  const wavStreamPlayer = wavStreamPlayerRef.value;

  // Set state variables
  startTimeRef.value = new Date().toISOString();
  isConnected.value = true;
  realtimeEvents.value = [];
  items.value = client.conversation.getItems();

  // Connect to microphone
  await wavRecorder.begin();

  // Connect to audio output
  await wavStreamPlayer.connect();

  // Connect to realtime API
  await client.connect();
  client.sendUserMessageContent([
    {
      type: `input_text`,
      text: `Hello!`,
      // text: `For testing purposes, I want you to list ten car brands. Number each item, e.g. "one (or whatever number you are one): the item name".`
    },
  ]);

  if (client.getTurnDetectionType() === 'server_vad') {
    await wavRecorder.record((data: any) => client.appendInputAudio(data.mono));
  }
};

// Method to disconnect and reset the conversation state
const disconnectConversation = async () => {
  isConnected.value = false;
  realtimeEvents.value = [];
  items.value = [];
  memoryKv.value = {};
  coords.value = { lat: 37.775593, lng: -122.418137 };
  marker.value = null;

  const client = clientRef.value;
  if (client) {
    client.disconnect();
  }

  const wavRecorder = wavRecorderRef.value;
  if (wavRecorder) {
    await wavRecorder.end();
  }

  const wavStreamPlayer = wavStreamPlayerRef.value;
  if (wavStreamPlayer) {
    await wavStreamPlayer.interrupt();
  }
};

// Method to start recording in push-to-talk mode
const startRecording = async () => {
  isRecording.value = true;
  const client = clientRef.value;
  const wavRecorder = wavRecorderRef.value;
  const wavStreamPlayer = wavStreamPlayerRef.value;

  // Interrupt audio output and handle track offset
  const trackSampleOffset = await wavStreamPlayer.interrupt();
  if (trackSampleOffset?.trackId) {
    const { trackId, offset } = trackSampleOffset;
    await client.cancelResponse(trackId, offset);
  }

  // Start recording and append input audio
  await wavRecorder.record((data: any) => client.appendInputAudio(data.mono));
};


// Method to stop recording in push-to-talk mode
const stopRecording = async () => {
  isRecording.value = false;
  const client = clientRef.value;
  const wavRecorder = wavRecorderRef.value;

  // Pause recording and create a response
  await wavRecorder.pause();
  client.createResponse();
};

// Method to delete a conversation item
const deleteConversationItem = async (id: string) => {
  const client = clientRef.value;
  if (client) {
    client.deleteItem(id);
  }
};

</script>

<template>
  <div data-component="ConsolePage">
    <div className="content-top">
      <div className="content-title">
        <img src="/openai-logomark.svg" />
        <span>realtime console</span>
      </div>
    </div>
    <div className="content-main">
      <div className="content-logs">
        <div className="content-block events">
          <div className="visualization">
            <div className="visualization-entry client">
              <canvas ref="clientCanvasRef" />
            </div>
            <div className="visualization-entry server">
              <canvas ref="serverCanvasRef" />
            </div>
          </div>
          <div className="content-block-title">events</div>
          <div class="content-block-body" ref="eventsScrollRef">
            <!-- Show message when no events are available -->
            <div v-if="!realtimeEvents.length">awaiting connection...</div>

            <!-- Render realtime events -->
            <div v-for="(realtimeEvent, index) in realtimeEvents" :key="realtimeEvent.event.event_id" class="event">
              <div class="event-timestamp">
                {{ formatTime(realtimeEvent.time) }}
              </div>
              <div class="event-details">
                <div class="event-summary" @click="toggleEventDetails(realtimeEvent.event.event_id)">
                  <div class="event-source" :class="{
                    error: realtimeEvent.event.type === 'error',
                    [realtimeEvent.source]: true,
                  }">
                    <FontAwesomeIcon v-if="realtimeEvent.source === 'client'" icon="arrow-up" />
                    <FontAwesomeIcon v-else icon="arrow-down" />
                    <span>
                      {{
                        realtimeEvent.event.type === 'error'
                          ? 'error!'
                          : realtimeEvent.source
                      }}
                    </span>
                  </div>
                  <div class="event-type">
                    {{ realtimeEvent.event.type }}
                    <span v-if="realtimeEvent.count"> ({{ realtimeEvent.count }})</span>
                  </div>
                </div>

                <!-- Expanded event payload -->
                <div v-if="expandedEvents[realtimeEvent.event.event_id]" class="event-payload">
                  <pre>{{ JSON.stringify(realtimeEvent.event, null, 2) }}</pre>
                </div>
              </div>
            </div>

          </div>
        </div>
        <div className="content-block conversation">
          <div className="content-block-title">conversation</div>
          <div class="content-block-body" data-conversation-content>
            <!-- Show message when no items are available -->
            <div v-if="!items.length">awaiting connection...</div>

            <!-- Render conversation items -->
            <div v-for="(conversationItem, index) in items" :key="conversationItem.id" class="conversation-item">
              <div class="speaker" :class="conversationItem.role || ''">
                <div>
                  {{
                    (conversationItem.role || conversationItem.type).replaceAll('_', ' ')
                  }}
                </div>
                <div class="close" @click="deleteConversationItem(conversationItem.id)">
                  <font-awesome-icon icon='x' />
                </div>
              </div>

              <div class="speaker-content">
                <!-- Tool response -->
                <div v-if="conversationItem.type === 'function_call_output'">
                  {{ conversationItem.formatted.output }}
                </div>

                <!-- Tool call -->
                <div v-if="conversationItem.formatted.tool">
                  {{ conversationItem.formatted.tool.name }}(
                  {{ conversationItem.formatted.tool.arguments }})
                </div>

                <!-- User role -->
                <div v-if="!conversationItem.formatted.tool && conversationItem.role === 'user'">
                  {{
                    conversationItem.formatted.transcript ||
                    (conversationItem.formatted.audio?.length
                      ? '(awaiting transcript)'
                      : conversationItem.formatted.text || '(item sent)')
                  }}
                </div>

                <!-- Assistant role -->
                <div v-if="!conversationItem.formatted.tool && conversationItem.role === 'assistant'">
                  {{
                    conversationItem.formatted.transcript ||
                    conversationItem.formatted.text || '(truncated)'
                  }}
                </div>

                <!-- File playback -->
                <audio v-if="conversationItem.formatted.file" :src="conversationItem.formatted.file.url" controls />
              </div>
            </div>
          </div>
        </div>
        <div className="content-actions">
          <Toggle :defaultValue="false" :labels="['manual', 'vad']" :values="['none', 'server_vad']"
            @change="onToggleChange" />
          <div class="spacer"></div>
          <div class="spacer"></div>
          <div v-if="isConnected && canPushToTalk">
            <Button :label="isRecording ? 'release to send' : 'push to talk'"
              :buttonStyle="isRecording ? 'alert' : 'regular'" :disabled="!isConnected || !canPushToTalk"
              @mousedown="startRecording" @mouseup="stopRecording" />
          </div>
          <div class="spacer"></div>
          <Button :label="isConnected ? 'disconnect' : 'connect'" :iconPosition="isConnected ? 'end' : 'start'"
            :buttonStyle="isConnected ? 'regular' : 'action'" :icon="isConnected ? 'x' : 'bolt'"
            @click="isConnected ? disconnectConversation() : connectConversation()" />
        </div>
      </div>
      <div className="content-right">
        <div className="content-block map">
          <div className="content-block-title">get_weather()</div>
          <div className="content-block-title bottom">
            {{ marker?.location || 'not yet retrieved' }}
            <!-- Display temperature if available -->
            <template v-if="marker?.temperature">
              <br />
              🌡️ {{ marker.temperature.value }} {{ marker.temperature.units }}
            </template>
            <!-- Display wind speed if available -->
            <template v-if="marker?.wind_speed">
              🍃 {{ marker.wind_speed.value }} {{ marker.wind_speed.units }}
            </template>
          </div>
          <div className="content-block-body full">
            <Map v-if="coords" :center="[coords.lat, coords.lng]" :location="coords.location" />
          </div>
        </div>
        <div className="content-block kv">
          <div className="content-block-title">set_memory()</div>
          <div className="content-block-body content-kv">
            {{ formattedMemoryKv }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="scss">
@import './ConsolePage.scss';
/* 引入现成的 SCSS 文件 */
</style>
