<script>
  import { onMount } from 'svelte';
  import { faGoogle, faMicrosoft, faYahoo } from '@fortawesome/free-brands-svg-icons';
  import { faFileDownload } from '@fortawesome/free-solid-svg-icons';
  import { FontAwesomeIcon } from '@fortawesome/svelte-fontawesome';

  const today = new Date().toISOString().substr(0, 10);

  let title = "Event Title";
  let date = today;
  let time = "12:00";
  let description = "Event Description";
  let location = "Event Location";
  let isReadOnly = false;
  let timeZone = Intl.DateTimeFormat().resolvedOptions().timeZone;
  let localDateTime = "";
  let notificationVisible = false;

  onMount(() => {
    const params = new URLSearchParams(window.location.search);
    if (params.get('mode') === 'readonly') {
      isReadOnly = true;
      title = params.get('title') || title;
      date = params.get('date') || date;
      time = params.get('time') || time;
      description = params.get('description') || description;
      location = params.get('location') || location;
      timeZone = params.get('timezone') || timeZone;

      // Convert to local date and time
      const eventStart = new Date(`${date}T${time}`);
      localDateTime = eventStart.toLocaleString(); // Format date and time to user's locale
    }
  });

  $: shareableURL = generateURL(title, date, time, description, location, 'readonly', timeZone);
  $: calendarUrls = generateCalendarURLs(title, date, time, description, location, timeZone);

  function generateURL(title, date, time, description, location, mode, timeZone) {
    const params = new URLSearchParams({
      title,
      date,
      time,
      description,
      location,
      timezone: timeZone
    });
    if (mode) {
      params.append('mode', mode);
    }
    return `${window.location.origin}?${params.toString()}`;
  }

  function generateCalendarURLs(title, date, time, description, location, timeZone) {
    const eventStart = new Date(`${date}T${time}`);
    const eventEnd = new Date(eventStart.getTime() + 60 * 60 * 1000); // Assume 1 hour duration

    const formatDate = (date) => date.toISOString().replace(/-|:|\.\d+/g, '');

    const googleCalendarUrl = `https://www.google.com/calendar/render?action=TEMPLATE&text=${encodeURIComponent(title)}&dates=${formatDate(eventStart)}/${formatDate(eventEnd)}&details=${encodeURIComponent(description)}&location=${encodeURIComponent(location)}&ctz=${encodeURIComponent(timeZone)}`;
    
    const outlookCalendarUrl = `https://outlook.live.com/calendar/deeplink/compose?subject=${encodeURIComponent(title)}&startdt=${eventStart.toISOString()}&enddt=${eventEnd.toISOString()}&body=${encodeURIComponent(description)}&location=${encodeURIComponent(location)}`;

    const yahooCalendarUrl = `https://calendar.yahoo.com/?v=60&title=${encodeURIComponent(title)}&st=${formatDate(eventStart)}&et=${formatDate(eventEnd)}&desc=${encodeURIComponent(description)}&in_loc=${encodeURIComponent(location)}`;

    return {
      google: googleCalendarUrl,
      outlook: outlookCalendarUrl,
      yahoo: yahooCalendarUrl,
    };
  }

  function downloadICSFile() {
    const eventStart = new Date(`${date}T${time}`);
    const eventEnd = new Date(eventStart.getTime() + 60 * 60 * 1000); // Assume 1 hour duration

    const formatDate = (date) => date.toISOString().replace(/-|:|\.\d+/g, '').slice(0, -1);

    const icsContent = `BEGIN:VCALENDAR
VERSION:2.0
BEGIN:VEVENT
SUMMARY:${title}
DESCRIPTION:${description}
LOCATION:${location}
DTSTART:${formatDate(eventStart)}
DTEND:${formatDate(eventEnd)}
END:VEVENT
END:VCALENDAR`;

    const blob = new Blob([icsContent], { type: 'text/calendar' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `${title}.ics`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  }

  function copyToClipboard(text) {
    navigator.clipboard.writeText(text).then(() => {
      notificationVisible = true;
      setTimeout(() => {
        notificationVisible = false;
      }, 2000);
    });
  }

  function resetToEditable() {
    window.location.href = window.location.origin;
  }
</script>

<main>
  <h1>Calendar Invite</h1>

  {#if isReadOnly}
    <div class="readonly-container">
      <h2>{title}</h2>
      <p><strong>Date:</strong> {date}</p>
      <p><strong>Time:</strong> {time} ({timeZone})</p>
      <p><strong>Your Local Time:</strong> {localDateTime}</p> <!-- Display the local date and time -->
      <p><strong>Description:</strong> {description}</p>
      <p><strong>Location:</strong> {location}</p>
      <h2>Add to your Calendar</h2>
      <a href={calendarUrls.google} target="_blank" class="calendar-link google">
        <FontAwesomeIcon icon={faGoogle} class="calendar-logo" /> Google Calendar
      </a>
      <a href={calendarUrls.outlook} target="_blank" class="calendar-link outlook">
        <FontAwesomeIcon icon={faMicrosoft} class="calendar-logo" /> Outlook Calendar
      </a>
      <a href={calendarUrls.yahoo} target="_blank" class="calendar-link yahoo">
        <FontAwesomeIcon icon={faYahoo} class="calendar-logo" /> Yahoo Calendar
      </a>
      <button on:click={downloadICSFile} class="calendar-link ics">
        <FontAwesomeIcon icon={faFileDownload} class="calendar-logo" /> Download .ics File
      </button>
      <button on:click={resetToEditable} class="generate-new-link">Generate your own calendar link</button>
    </div>
  {:else}
    <form>
      <label>
        Event Title:
        <input type="text" bind:value={title} placeholder="Enter event title" />
      </label>
      <label>
        Date:
        <input type="date" bind:value={date} />
      </label>
      <label>
        Time:
        <input type="time" bind:value={time} />
      </label>
      <label>
        Description:
        <textarea bind:value={description} placeholder="Enter event description"></textarea>
      </label>
      <label>
        Location:
        <input type="text" bind:value={location} placeholder="Enter event location" />
      </label>
    </form>

    <div class="output-container">
      <h2>Your Shareable URL</h2>
      <p class="url">{shareableURL}</p>
      <div class="copy-container">
        <button on:click={() => copyToClipboard(shareableURL)}>Copy URL</button>
        {#if notificationVisible}
          <div class="notification">URL copied!</div>
        {/if}
      </div>
    </div>
  {/if}
</main>

<style>
  :global(body) {
    margin: 0;
    font-family: 'Roboto', sans-serif;
    background-color: #f0f2f5;
    color: #333;
  }

  main {
    padding: 20px;
    max-width: 600px;
    margin: auto;
    background-color: #ffffff;
    border-radius: 10px;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
    transition: box-shadow 0.3s ease;
    position: relative;
  }

  main:hover {
    box-shadow: 0 6px 25px rgba(0, 0, 0, 0.15);
  }

  h1 {
    text-align: center;
    color: #3a3d42;
    font-size: 1.8em;
    margin-bottom: 20px;
  }

  form {
    display: flex;
    flex-direction: column;
    gap: 15px;
    margin-bottom: 20px;
  }

  label {
    font-weight: bold;
    color: #555;
  }

  input, textarea {
    padding: 10px;
    font-size: 16px;
    border: 1px solid #ddd;
    border-radius: 5px;
    width: 100%;
    box-sizing: border-box;
  }

  textarea {
    resize: vertical;
    min-height: 80px;
  }

  .output-container {
    margin-top: 30px;
    text-align: center;
  }

  .url {
    word-break: break-all;
    background-color: #e7f3ff;
    padding: 10px;
    border-radius: 5px;
    margin-bottom: 15px;
    font-family: monospace;
    color: #0056b3;
  }

  button {
    background-color: #007bff;
    color: white;
    padding: 10px 20px;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    font-size: 16px;
    transition: background-color 0.3s ease;
    position: relative;
  }

  button:hover {
    background-color: #0056b3;
  }

  .calendar-link {
    display: inline-block;
    margin: 10px 5px;
    padding: 10px 20px;
    border-radius: 5px;
    color: white;
    text-decoration: none;
    font-size: 16px;
    transition: background-color 0.3s ease;
  }

  .google {
    background-color: #34a853;
  }

  .google:hover {
    background-color: #0a8f3c;
  }

  .outlook {
    background-color: #0072c6;
  }

  .outlook:hover {
    background-color: #005fa3;
  }

  .yahoo {
    background-color: #410093;
  }

  .yahoo:hover {
    background-color: #330075;
  }

  .ics {
    background-color: #f39c12;
  }

  .ics:hover {
    background-color: #e67e22;
  }

  .readonly-container {
    text-align: center;
    padding: 20px;
    background-color: #f9f9f9;
    border-radius: 10px;
    border: 1px solid #ddd;
  }

  .readonly-container h2 {
    color: #333;
  }

  .readonly-container p {
    margin: 10px 0;
    color: #555;
  }

  .readonly-container .generate-new-link {
    margin-top: 20px;
    background-color: #007bff;
  }

  .readonly-container .generate-new-link:hover {
    background-color: #0056b3;
  }

  .copy-container {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;
    margin-top: 10px;
  }

  .notification {
    padding: 5px 10px;
    background-color: #4caf50;
    color: white;
    border-radius: 5px;
    animation: fadeInOut 2s ease-in-out;
    font-size: 14px;
  }

  @keyframes fadeInOut {
    0% {
      opacity: 0;
    }
    10% {
      opacity: 1;
    }
    90% {
      opacity: 1;
    }
    100% {
      opacity: 0;
    }
  }

  @media (max-width: 600px) {
    main {
      padding: 15px;
    }

    .calendar-link {
      width: 100%;
      margin: 10px 0;
      text-align: center;
    }
  }
</style>
