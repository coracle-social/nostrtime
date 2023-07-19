<script lang="ts">
  import cx from "classnames"
  import {fly, fade} from "svelte/transition"
  import NDK, {NDKNip07Signer, NDKEvent} from "@nostr-dev-kit/ndk"

  const now = () => Math.round(Date.now() / 1000)

  const changeDay = (d, x) => {
    const date = new Date(d)

    date.setDate(d.getDate() + x)

    return date
  }

  const dateToSeconds = date => Math.round(date.valueOf() / 1000)

  const generateMonth = date => ({
    date,
    days: getDaysInMonth(date.getFullYear(), date.getMonth()),
  })


  const getTimezoneOffset = () => new Date().getTimezoneOffset() * 60000

  const fromNostrTime = seconds => new Date(seconds * 1000 - getTimezoneOffset()).toISOString().slice(0, -5).split("T")

  const toNostrTime = (date, time) => dateToSeconds(new Date(`${date}T${time}`)).toString()

  function getDaysInMonth(year, month) {
    const days = []
    const firstDay = new Date(year, month, 1).getDay()
    const daysInThisMonth = new Date(year, month + 1, 0).getDate()
    const daysInLastMonth = new Date(year, month, 0).getDate()
    const prevMonth = month == 0 ? 11 : month - 1

    //  show the days before the start of this month (disabled) - always less than 7
    for (let i = daysInLastMonth - firstDay; i < daysInLastMonth; i++) {
      days.push(new Date(prevMonth === 11 ? year - 1 : year, prevMonth, i + 1))
    }

    //  show the days in this month (enabled) - always 28 - 31
    for (let i = 0; i < daysInThisMonth; i++) {
      days.push(new Date(year, month, i + 1))
    }

    //  show any days to fill up the last row (disabled) - always less than 7
    for (let i = 0; days.length % 7; i++) {
      days.push(new Date(month == 11 ? year + 1 : year, (month + 1) % 12, i + 1))
    }

    return days
  }

  const getMeta = ({tags}) => {
    const meta = {}

    for (const [k, v] of tags) {
      meta[k] = v
    }

    return meta
  }

  const eventIsInRange = (date, event) => {
    const meta = getMeta(event)
    const startOfDay = dateToSeconds(date.setHours(0, 0, 0, 0))
    const endOfDay = dateToSeconds(date.setHours(23, 59, 59, 59))

    return parseInt(meta.start) <= endOfDay && parseInt(meta.end) >= startOfDay
  }

  const getDateEvents = date =>
    events
      .filter(e => eventIsInRange(date, e))
      .sort((a, b) => (getMeta(a).start > getMeta(b).start ? 1 : 0))

  const pendingNdk = (async () => {
    const ndk = new NDK({
      explicitRelayUrls: [
        "wss://nos.lol",
        "wss://nostr-pub.wellorder.net",
        "wss://relay.nostr.band",
      ],
    })

    await ndk.connect()

    return ndk
  })()

  let user
  let draft
  let key = Math.random()
  let events = []
  let month = generateMonth(new Date())

  const prevMonth = () => {
    const y = month.date.getMonth() === 0 ? month.date.getFullYear() - 1 : month.date.getFullYear()
    const m = month.date.getMonth() === 0 ? 11 : month.date.getMonth() - 1

    month = generateMonth(new Date(y, m, 1))
  }

  const nextMonth = () => {
    const y = month.date.getMonth() === 11 ? month.date.getFullYear() + 1 : month.date.getFullYear()
    const m = month.date.getMonth() === 11 ? 0 : month.date.getMonth() + 1

    month = generateMonth(new Date(y, m, 1))
  }

  const login = async () => {
    const ndk = await pendingNdk

    ndk.signer = new NDKNip07Signer()

    await ndk.signer.blockUntilReady()

    user = await ndk.signer.user()

    loadCalendar()
  }

  const loadCalendar = async () => {
    const ndk = await pendingNdk

    const filter = {kinds: [31923], limit: 1000}

    const allEvents = Array.from(await ndk.fetchEvents(filter))

    const deletions = Array.from(
      await ndk.fetchEvents({
        kinds: [5],
        "#a": Array.from(allEvents).map(e => e.tagReference()[1]),
      }),
    )

    const deletedAddresses = new Set(deletions.map(e => e.tagValue("a")))

    events = allEvents.filter(e => !deletedAddresses.has(e.tagReference()[1]))
    key = Math.random()
  }

  const newEvent = () => {
    const [date, time] = fromNostrTime(now())

    draft = {
      id: crypto.randomUUID(),
      name: "My Event",
      content: "An example event",
      startDate: date,
      startTime: time,
      endDate: date,
      endTime: time,
    }
  }

  const publishEvent = async () => {
    const ndk = await pendingNdk
    const event = new NDKEvent(ndk, {
      kind: 31923,
      content: draft.content,
      tags: [
        ["d", draft.id],
        ["name", draft.name],
        ["start", toNostrTime(draft.startDate, draft.startTime)],
        ["end", toNostrTime(draft.endDate, draft.endTime)],
      ],
    })

    await event.publish()

    events = events.filter(e => getMeta(e).d !== draft.id).concat(event)
    key = Math.random()

    draft = null
  }

  const editEvent = async e => {
    const meta = getMeta(e)
    const [startDate, startTime] = fromNostrTime(meta.start)
    const [endDate, endTime] = fromNostrTime(meta.end)

    draft = {
      event: e,
      id: meta.d,
      name: meta.name,
      content: e.content,
      startDate,
      startTime,
      endDate,
      endTime,
    }
  }

  const clearEvent = async () => {
    draft = null
  }

  const deleteEvent = async () => {
    if (confirm("Are you sure you want to delete this event?")) {
      await draft.event.delete()

      events = events.filter(e => getMeta(e).d !== draft.id)
      key = Math.random()

      draft = null
    }
  }

  loadCalendar()
</script>

<div class="absolute inset-0 flex flex-col">
  <div class="p-4 flex justify-between grid grid-cols-3">
    <h1 class="text-2xl">NostrTime</h1>
    <div class="flex justify-center items-center gap-2">
      <span class="p-2 pt-3 cursor-pointer text-xs" on:click={prevMonth}>◀</span>
      <span class="font-bold w-36 text-center">
        {month.date.toLocaleString("default", {month: "long", year: "numeric"})}
      </span>
      <span class="p-2 pt-3 cursor-pointer text-xs" on:click={nextMonth}>▶</span>
    </div>
    <div class="flex justify-end">
      {#if user}
        <button class={`padding button1`} on:click={newEvent}> Add Event </button>
      {:else}
        <button class={`padding button1`} on:click={login}> Log In </button>
      {/if}
    </div>
  </div>
  <div class="grid grid-cols-7 mx-4 card-default-t card-default-l">
    {#key key}
      {#each month.days as date, i}
        <div class={`aspect-square text-gray-500 text-xs relative`}>
          <div class={`absolute inset-0 card-default-r card-default-b`} />
          <div class="p-1">
            {date.getDate()}
            <div class="flex flex-col gap-1">
              {#each getDateEvents(date) as event}
                {@const meta = getMeta(event)}
                {@const isContinuation = eventIsInRange(changeDay(date, -1), event)}
                {@const isContinued = eventIsInRange(changeDay(date, 1), event)}
                {@const isOwn = event.pubkey === user?.hexpubkey()}
                <div
                  class={cx(
                    "relative z-10 cursor-pointer p-1 whitespace-nowrap",
                    {
                      "z-20": (!isContinuation || date.getDay() === 0) && isContinued,
                      "bg-white text-blue-500 border border-solid border-blue-500": !isOwn,
                      "bg-blue-500 text-white": isOwn,
                      "-ml-1 border-l-0": isContinuation,
                      "rounded-s": !isContinuation,
                      "-mr-1 border-r-0": isContinued,
                      "rounded-e text-ellipsis overflow-hidden": !isContinued,
                      "text-white/0": isContinuation && date.getDay() !== 0,
                    },
                  )}
                  on:click={() => editEvent(event)}>
                  {meta.name}
                </div>
              {/each}
            </div>
          </div>
        </div>
      {/each}
    {/key}
  </div>
  <small class="p-2 text-gray-400 text-center"> NostrTime is powered by NIP-52. </small>
  {#if draft}
    {@const isEditable = (!draft.event && user) || draft.event?.pubkey === user?.hexpubkey()}
    <div
      transition:fade
      class="cursor-pointer fixed z-20 inset-0 bg-black/50 p-4"
      on:click={clearEvent}>
      <div
        in:fly={{y: 20}}
        class="cursor-auto bg-white rounded-xl p-4 flex flex-col gap-2"
        on:click|stopPropagation>
        <h2 class="text-xl">Event Details</h2>
        <div class="grid grid-cols-3 items-center gap-2">
          <label for="name">Name</label>
          <input
            disabled={!isEditable}
            name="name"
            type="text"
            class={`padding card-default col-span-2 rounded-xl`}
            bind:value={draft.name} />
          <label for="description">Description</label>
          <input
            disabled={!isEditable}
            name="description"
            type="text"
            class={`padding card-default col-span-2 rounded-xl`}
            bind:value={draft.content} />
          <label for="start">Start</label>
          <input
            disabled={!isEditable}
            name="startDate"
            type="date"
            class={`padding card-default col-span-2 sm:col-span-1 rounded-xl`}
            bind:value={draft.startDate} />
          <div class="col-span-1 sm:hidden" />
          <input
            disabled={!isEditable}
            name="startTime"
            type="time"
            class={`padding card-default col-span-2 sm:col-span-1 rounded-xl`}
            bind:value={draft.startTime} />
          <label for="end">End</label>
          <input
            disabled={!isEditable}
            name="endDate"
            type="date"
            class={`padding card-default col-span-2 sm:col-span-1 rounded-xl`}
            bind:value={draft.endDate} />
          <div class="col-span-1 sm:hidden" />
          <input
            disabled={!isEditable}
            name="endTime"
            type="time"
            class={`padding card-default col-span-2 sm:col-span-1 rounded-xl`}
            bind:value={draft.endTime} />
        </div>
        {#if isEditable}
          <div class="flex justify-end gap-2">
            {#if draft.event}
              <button class={`padding button3`} on:click={deleteEvent}> Delete </button>
            {/if}
            <button class={`padding button1`} on:click={publishEvent}> Save Event </button>
          </div>
        {/if}
      </div>
    </div>
  {/if}
</div>
