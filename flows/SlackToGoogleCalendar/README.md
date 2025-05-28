# n8n Slack Calendar Assistant - Complete Implementation Guide

## 📋 System Overview

This is an n8n-based automation workflow that can:
- Monitor Slack channel messages
- Use Azure OpenAI to intelligently analyze schedule information in messages
- Automatically create Google Calendar events
- Decide whether to create automatically or require confirmation based on confidence levels
- Support both all-day and timed events
- Provide detailed Slack notification feedback

## 🏗️ System Architecture

```
Slack Message → Webhook → Filter Channel Messages → Basic LLM Chain (Azure OpenAI) 
    ↓
Process AI Response → Confidence Filter → Event Type Filter
    ↓                    ↓                    ↓
Error Notification  High Confidence    Low Confidence
                         ↓                    ↓
                   Create Calendar    Confirmation Request
                         ↓
                   Success Notification
```

## 🔧 Node Configuration Details

### 1. Webhook Node
**Name**: `Webhook`
**Type**: Webhook Trigger

**Configuration**:
```yaml
HTTP Method: POST
Path: slack-calendar
Response Mode: Respond With
Response Data: ={{$json.challenge || 'OK'}}
```

**Purpose**: Receive webhook requests from Slack Event Subscriptions

---

### 2. Filter Channel Messages Node
**Name**: `Filter Channel Messages`
**Type**: IF

**Key Configuration**:
```javascript
// Condition 1: Check channel
Value 1: ={{$json.channel}}
Operation: Equal
Value 2: C08NVUQUK8F  // Replace with your actual channel ID

// Condition 2: Check message type
Value 1: ={{$json.type}}
Operation: Equal
Value 2: message

// Condition 3: Exclude Bot messages (prevent infinite loop)
Value 1: ={{$json.user}}
Operation: Not Equal
Value 2: YOUR_BOT_USER_ID  // Replace with your Bot User ID

// Condition 4: Exclude system messages
Value 1: ={{$json.subtype}}
Operation: Is Empty

// Condition 5: Exclude reply messages
Value 1: ={{$json.thread_ts}}
Operation: Is Empty

// Logic
Combine: AND
```

**Purpose**: Filter to process only user messages from target channel, avoid infinite loops

---

### 3. Basic LLM Chain Node
**Name**: `AI Message Analyzer`
**Type**: Basic LLM Chain

**Language Model Settings**:
- Model: Azure OpenAI Chat Model
- Credential: Azure OpenAI API
- Deployment Name: Your Azure deployment name (e.g., gpt-4)

**Complete Prompt**:
```
You are a professional calendar management assistant. Please analyze user's Slack messages and extract possible schedule information.

🔍 Location Detection Rules (Important):
1. Locations after movement verbs:
   - "go to XX", "to XX", "towards XX", "attend XX"
   - "travel to XX", "depart to XX", "fly to XX"
   - Example: "go to Nagoya for fun" → location: "Nagoya"

2. Locations after position prepositions:
   - "at XX", "in XX", "located at XX"
   - "held at XX", "conducted at XX", "convened at XX"
   - Example: "meeting at Taipei" → location: "Taipei"

3. Common location types:
   - Cities/Countries: Taipei, Tokyo, Nagoya, Singapore, USA, Japan
   - Landmarks/Attractions: Taipei 101, Disneyland, National Palace Museum, Mount Fuji
   - Venues: Conference Room A, cafe, restaurant, office, home
   - Online: Zoom, Teams, Google Meet, video call, online
   - Buildings: XX Building, XX Center, XX Hall, XX Auditorium

4. Compound location expressions:
   - "Taipei 101" → location: "Taipei 101"
   - "Hsinchu Science Park" → location: "Hsinchu Science Park"
   - "Conference Room A" → location: "Conference Room A"
   - "online meeting" → location: "online"

5. Location extraction priority:
   - Specific address > Building name > City name > Area name
   - If multiple locations, choose the most specific one

6. Special case handling:
   - "XX for fun", "XX travel", "XX business trip" → XX is location
   - "return to XX", "back to XX" → XX is location
   - "meet at XX", "dine at XX" → XX might be location

All-day event identification rules:
1. Travel activities: "go to XX for fun", "XX travel", "to XX", "XX business trip"
2. Holidays/Leave: "take leave", "vacation", "holiday"
3. Festivals/Celebrations: "birthday", "festival", "event day"
4. Workshops/Courses: "workshop", "training camp", "boot camp"
5. Activities with only date mentioned without specific time

Timed event identification rules:
1. Meetings: "meeting", "conference", "discussion"
2. Presentations: "presentation", "report"
3. Activities with specific time: "2 PM", "morning", "evening"

Time format rules:
- All-day events: startDateTime set to "YYYY-MM-DD" (no time component)
- Timed events: Use full ISO 8601 format "YYYY-MM-DDTHH:mm:ss+08:00"

Analysis rules:
1. Time keywords: tomorrow, next week, today, specific dates, specific times
2. Activity keywords: meeting, discussion, presentation, training, workshop, activity, fun, travel
3. Location information: conference room, address, online, video call, city names, building names
4. Participants: @username, titles, departments
5. Time inference:
   - "tomorrow afternoon" → tomorrow 14:00-15:00 (timed)
   - "next Wednesday meeting" → next Wednesday 09:00-10:00 (timed)
   - "2 o'clock meeting" → 14:00-15:00 (timed)
   - "go to XX for fun" → all-day event (pure date format)
   - "XX workshop" → all-day event (pure date format)
   - "take leave" → all-day event (pure date format)

Please reply in the following JSON format, do not add any other text:
{
  "hasEvent": true,
  "events": [
    {
      "title": "Event Title",
      "description": "Detailed description",
      "startDateTime": "2025-05-20",
      "endDateTime": "2025-05-20",
      "isAllDay": true,
      "location": "Specific location name",
      "attendees": ["email1@example.com"],
      "confidence": 0.95
    }
  ],
  "reasoning": "Analysis reason, including location identification logic"
}

Location detection examples:
- "5/20 go to Nagoya for fun" → location: "Nagoya" (extracted from "go to XX for fun")
- "meeting at Conference Room A tomorrow" → location: "Conference Room A" (extracted from "at XX")
- "business trip to Taipei next week" → location: "Taipei" (extracted from "to XX business trip")
- "6/15 Tokyo Disneyland day trip" → location: "Tokyo Disneyland" (compound location)
- "online meeting to discuss project" → location: "online" (virtual location)
- "work from home" → location: "home" (home location)
- "visit Hsinchu Science Park" → location: "Hsinchu Science Park" (park location)

Activity type and location examples:
- "5/20 go to Nagoya for fun" → all-day event, startDateTime: "2025-05-20", endDateTime: "2025-05-20", isAllDay: true, location: "Nagoya"
- "presentation at Conference Room B tomorrow 2 PM" → timed event, startDateTime: "2025-05-21T14:00:00+08:00", endDateTime: "2025-05-21T15:00:00+08:00", isAllDay: false, location: "Conference Room B"
- "6/15 Taipei 101 workshop" → all-day event, startDateTime: "2025-06-15", endDateTime: "2025-06-15", isAllDay: true, location: "Taipei 101"
- "online meeting next Wednesday 10 AM" → timed event, startDateTime: "2025-06-04T10:00:00+08:00", endDateTime: "2025-06-04T11:00:00+08:00", isAllDay: false, location: "online"

Confidence assessment:
- 0.9+: Complete time + location + activity type clear
- 0.7-0.9: Clear time and activity, location can be inferred
- 0.5-0.7: Vague time or activity type, location unclear
- <0.5: Lack of key information

Default settings:
- Time zone: Asia/Taipei (+08:00)
- Default timed event duration: 1 hour
- Meeting without specific time: 09:00-10:00
- Travel/vacation/workshop etc.: all-day events
- Location extraction: Prioritize extracting the most specific location information

Now please analyze the following Slack message: "={{$json.text}}"

⚠️ Important: Please pay special attention to location information extraction, ensuring no location-related words are missed.
```

**Purpose**: Use Azure OpenAI to analyze Slack messages and extract schedule information

---

### 4. Process AI Response Node
**Name**: `Process AI Response`
**Type**: Function

**Complete JavaScript Code**:
```javascript
// ===============================================
// Process AI Response - Complete JavaScript Code
// ===============================================

console.log('🚀 Starting AI response processing...');

// 1. Check basic input
const inputData = $input.first();
if (!inputData || !inputData.json) {
  console.log('❌ No input data, stopping execution');
  return [];
}

console.log('📥 Input data:', JSON.stringify(inputData.json, null, 2));

// 2. Get AI output (from text field)
const aiResponseText = inputData.json.text || inputData.json.output;
if (!aiResponseText) {
  console.log('❌ AI has no text output, stopping execution');
  return [];
}

console.log('🤖 AI raw response:', aiResponseText);

// 3. Parse JSON string
let aiOutput;
try {
  // Clean possible markdown format
  const cleanResponse = aiResponseText.replace(/```json\n?|```\n?/g, '').trim();
  aiOutput = JSON.parse(cleanResponse);
  console.log('✅ Parsed AI output:', JSON.stringify(aiOutput, null, 2));
} catch (error) {
  console.log('❌ JSON parsing failed:', error.message);
  return [{
    json: {
      status: 'error',
      error: `AI response parsing failed: ${error.message}`,
      rawResponse: aiResponseText,
      slackUser: 'unknown',
      slackChannel: 'unknown',
      slackTimestamp: Date.now().toString()
    }
  }];
}

// 4. Check Slack original data
let slackData = {};
try {
  // Try multiple ways to get Slack data
  if ($node && $node[0] && $node[0].json) {
    slackData = $node[0].json;
  } else if ($('Webhook') && $('Webhook').first()) {
    slackData = $('Webhook').first().json;
  } else if ($('Slack Webhook') && $('Slack Webhook').first()) {
    slackData = $('Slack Webhook').first().json;
  } else {
    // If cannot get original data, use defaults
    slackData = {
      text: 'Unable to get original message',
      user: 'unknown',
      channel: 'unknown',
      ts: Date.now().toString()
    };
  }
  console.log('📱 Slack data:', JSON.stringify(slackData, null, 2));
} catch (error) {
  console.log('⚠️ Error getting Slack data:', error.message);
  slackData = {
    text: 'Unable to get original message',
    user: 'unknown', 
    channel: 'unknown',
    ts: Date.now().toString()
  };
}

// 5. Check if AI found events
if (!aiOutput.hasEvent || !aiOutput.events || aiOutput.events.length === 0) {
  console.log('ℹ️ AI found no events');
  return [{
    json: {
      status: 'no_event',
      message: 'No valid schedule information found',
      originalMessage: slackData.text,
      slackUser: slackData.user,
      slackChannel: slackData.channel,
      slackTimestamp: slackData.ts,
      reasoning: aiOutput.reasoning || 'Unable to identify schedule-related content'
    }
  }];
}

// 6. Process found events (support all-day activities)
console.log(`📅 Found ${aiOutput.events.length} events, starting processing`);

try {
  const processedEvents = aiOutput.events.map((event, index) => {
    console.log(`🔄 Processing event ${index}:`, JSON.stringify(event, null, 2));
    
    // Validate if event has required fields
    if (!event.title && !event.startDateTime) {
      console.log(`⚠️ Event ${index} missing required fields, skipping`);
      return null;
    }

    // 🎯 Handle all-day and timed events
    let startDateTime, endDateTime, isAllDay = false;
    let calendarStartDateTime, calendarEndDateTime;
    
    try {
      // Check if it's an all-day event
      const isAllDayEvent = event.isAllDay === true || 
                           (!event.startDateTime.includes('T') && !event.endDateTime.includes('T'));
      
      if (isAllDayEvent) {
        // 📅 All-day event processing
        console.log(`📅 Event ${index} is all-day event`);
        isAllDay = true;
        
        // Ensure correct date format (YYYY-MM-DD)
        startDateTime = event.startDateTime.includes('T') 
          ? event.startDateTime.split('T')[0] 
          : event.startDateTime;
        endDateTime = event.endDateTime.includes('T') 
          ? event.endDateTime.split('T')[0] 
          : event.endDateTime;
          
        // Google Calendar all-day event format: pure date string
        calendarStartDateTime = startDateTime;
        calendarEndDateTime = endDateTime;
        
        // Validate date format
        if (!startDateTime.match(/^\d{4}-\d{2}-\d{2}$/)) {
          throw new Error('All-day event date format error');
        }
        
        console.log(`✅ All-day event time: ${startDateTime} to ${endDateTime}`);
        
      } else {
        // ⏰ Timed event processing
        console.log(`⏰ Event ${index} is timed event`);
        isAllDay = false;
        
        const startDate = new Date(event.startDateTime);
        const endDate = new Date(event.endDateTime);
        
        if (isNaN(startDate.getTime()) || isNaN(endDate.getTime())) {
          throw new Error('Invalid date time format');
        }
        
        // Ensure end time is after start time
        if (endDate <= startDate) {
          endDate.setTime(startDate.getTime() + 60 * 60 * 1000);
        }
        
        startDateTime = startDate.toISOString();
        endDateTime = endDate.toISOString();
        
        // Google Calendar timed event format: complete ISO string
        calendarStartDateTime = startDateTime;
        calendarEndDateTime = endDateTime;
        
        console.log(`✅ Timed event: ${startDateTime} to ${endDateTime}`);
      }
      
    } catch (dateError) {
      console.log(`⚠️ Event ${index} time processing failed, using default:`, dateError.message);
      
      // Default to tomorrow's all-day event
      const tomorrow = new Date();
      tomorrow.setDate(tomorrow.getDate() + 1);
      const dateStr = tomorrow.toISOString().split('T')[0];
      
      startDateTime = dateStr;
      endDateTime = dateStr;
      calendarStartDateTime = dateStr;
      calendarEndDateTime = dateStr;
      isAllDay = true;
    }
    
    // Process attendees list
    const attendees = (event.attendees || []).filter(email => 
      email && email.includes('@') && email.includes('.')
    );
    
    // Ensure confidence is float
    const confidence = parseFloat(event.confidence) || 0.5;
    console.log(`📊 Event ${index} confidence: ${confidence} (${Math.round(confidence * 100)}%)`);
    
    // Build detailed description
    const description = [
      event.description || '',
      '',
      '📋 Source Information:',
      `• Slack Message: ${slackData.text}`,
      `• Sender: <@${slackData.user}>`,
      `• Channel: <#${slackData.channel}>`,
      `• Event Type: ${isAllDay ? '📅 All-day Event' : '⏰ Timed Event'}`,
      `• AI Analysis Confidence: ${Math.round(confidence * 100)}%`,
      aiOutput.reasoning ? `• Analysis Explanation: ${aiOutput.reasoning}` : ''
    ].filter(line => line !== '').join('\n');
    
    // 🎯 Determine processing strategy based on confidence
    let eventStatus;
    if (confidence >= 0.7) {
      eventStatus = 'high_confidence';
      console.log(`✅ Event ${index} high confidence, auto processing`);
    } else {
      eventStatus = 'low_confidence';
      console.log(`⚠️ Event ${index} low confidence, needs confirmation`);
    }
    
    const processedEvent = {
      json: {
        eventIndex: index,
        title: event.title || 'Untitled Event',
        description: description,
        
        // 📅 Time information
        startDateTime: calendarStartDateTime,
        endDateTime: calendarEndDateTime,
        isAllDay: isAllDay,
        
        // 📍 Location and attendees
        location: event.location || '',
        attendees: attendees,
        
        // 📊 Analysis information
        confidence: confidence,
        reasoning: aiOutput.reasoning || '',
        
        // 📱 Original information
        originalMessage: slackData.text,
        slackUser: slackData.user,
        slackChannel: slackData.channel,
        slackTimestamp: slackData.ts,
        
        // 🔄 Processing status
        status: eventStatus
      }
    };
    
    console.log(`✅ Event ${index} processing complete:`, JSON.stringify(processedEvent.json, null, 2));
    return processedEvent;
  }).filter(event => event !== null);

  // 7. Check if there are valid processing results
  if (processedEvents.length === 0) {
    console.log('❌ No valid events to process');
    return [{
      json: {
        status: 'error',
        error: 'All events failed processing',
        originalMessage: slackData.text,
        slackUser: slackData.user,
        slackChannel: slackData.channel,
        slackTimestamp: slackData.ts
      }
    }];
  }

  console.log(`🎉 Successfully processed ${processedEvents.length} events`);
  console.log('📤 Final output:', JSON.stringify(processedEvents, null, 2));
  
  return processedEvents;
  
} catch (error) {
  console.log('❌ Error processing events:', error.message);
  console.log('🔍 Error stack:', error.stack);
  
  return [{
    json: {
      status: 'error',
      error: `Processing failed: ${error.message}`,
      originalMessage: slackData.text,
      slackUser: slackData.user,
      slackChannel: slackData.channel,
      slackTimestamp: slackData.ts,
      debugInfo: {
        errorStack: error.stack,
        inputData: inputData.json,
        aiOutput: aiOutput
      }
    }
  }];
}
```

**Purpose**: Parse AI output, handle time formats, classify events based on confidence

---

### 5. Confidence Filter Node
**Name**: `Confidence Filter`
**Type**: IF

**Configuration**:
```javascript
// Use single condition (recommended)
Value 1: ={{$json.status === 'high_confidence'}}
Operation: Equal
Value 2: true

// Or use separate conditions
// Condition 1: Status check
Value 1: ={{$json.status}}
Operation: Equal
Value 2: high_confidence

// Condition 2: Confidence check
Value 1: ={{parseFloat($json.confidence)}}
Operation: Greater than or equal
Value 2: 0.7

// Logic: AND
```

**Purpose**: Decide whether to auto-create or require confirmation based on confidence (≥70%)

---

### 6. Create Calendar Event Node
**Name**: `Create Calendar Event`
**Type**: Google Calendar

**Configuration**:
```javascript
Calendar ID: primary
Resource: Event
Operation: Create

// Basic fields
Summary: ={{$json.title}}
Start: ={{$json.startDateTime}}
End: ={{$json.endDateTime}}
Description: ={{$json.description}}
Location: ={{$json.location}}

// 🎯 Key setting: All Day Event
// Option 1: Use fixed value (most stable)
All Day Event: Yes  // Manual selection (if all are all-day events)

// Option 2: Use expression (needs dynamic handling)
All Day Event: ={{Boolean($json.isAllDay)}}

// Other settings
Send Notifications: Yes
Time Zone: Asia/Taipei
```

**Known Issues**: All Day Event field expressions may be unstable, recommend using branch handling

**Branch Solution**:
If All Day Event expression has issues, create two dedicated nodes:
1. `Create All Day Event` - All Day: Yes (fixed)
2. `Create Timed Event` - All Day: No (fixed)
Use Event Type Filter before to route

**Purpose**: Create Google Calendar events

---

### 7. Success Notification Node
**Name**: `Success Notification`
**Type**: Slack

**Configuration**:
```javascript
Credential: Slack API
Resource: Message
Operation: Post

// Channel setting
Select a Channel: By ID
Channel ID: ={{$json.slackChannel || 'C08NVUQUK8F'}}

// Message content
Text: 
✅ **Calendar event successfully created!**

📅 **{{$json.summary}}**
{{$json.start.date ? 
  '🗓️ ' + $json.start.date + ' (All-day event)' : 
  '🕐 Time: ' + $json.start.dateTime + ' - ' + $json.end.dateTime
}}
📍 Location: {{$json.location || 'Not specified'}}

🔗 [View Event]({{$json.htmlLink}})

_🤖 Automatically created by AI assistant_

// Thread reply setting
Thread TS: ={{$json.slackTimestamp}}
```

**Purpose**: Send successful event creation Slack notification

---

### 8. Low Confidence Notification Node
**Name**: `Low Confidence Notification`
**Type**: Slack

**Configuration**:
```javascript
Channel ID: ={{$json.slackChannel}}

Text:
⚠️ **Schedule information needs confirmation**

I found possible schedule information in your message, but it needs confirmation:

📝 **{{$json.title}}**
🕐 Time: {{DateTime.fromISO($json.startDateTime).toFormat('MM/dd HH:mm')}} - {{DateTime.fromISO($json.endDateTime).toFormat('HH:mm')}}
📍 Location: {{$json.location || 'Not specified'}}
📊 Confidence: {{Math.round($json.confidence * 100)}}%

💭 AI Analysis: _{{$json.reasoning}}_

Please reply with the following options:
✅ Confirm creation
❌ Cancel
✏️ Provide more details

Thread TS: ={{$json.slackTimestamp}}
```

**Purpose**: Low confidence events require user confirmation

---

### 9. Error Notification Node
**Name**: `Error Notification`
**Type**: Slack

**Configuration**:
```javascript
Channel ID: ={{$json.slackChannel}}

Text:
❌ **Processing failed**

{{$json.status === 'no_event' ? 'No clear schedule information found in your message.' : 'Error occurred while processing your message:'}}

{{$json.status === 'error' ? '🐛 Error details: ' + $json.error : ''}}
{{$json.reasoning ? '🤔 AI Analysis: ' + $json.reasoning : ''}}

💡 **Suggested format examples:**
• `Team meeting tomorrow 2 PM`
• `Client presentation 2025-06-01 14:00`
• `Project discussion next Wednesday 10 AM at Conference Room A`
• `All-day workshop 6/15`

Original message: _{{$json.originalMessage}}_

Thread TS: ={{$json.slackTimestamp}}
```

**Purpose**: Handle error and no-event situations notification

## 🔐 Credential Configuration

### 1. Slack API Credential
**Type**: Slack API
**Settings**:
```yaml
Access Token: xoxb-your-bot-token-here
```

**How to obtain**:
1. Go to https://api.slack.com/apps
2. Create new App or select existing App
3. OAuth & Permissions → Bot User OAuth Token

### 2. Azure OpenAI Credential
**Type**: Azure OpenAI
**Settings**:
```yaml
API Key: your-azure-openai-api-key
Resource Name: your-resource-name
API Version: 2024-02-15-preview
```

### 3. Google Calendar Credential
**Type**: Google Calendar OAuth2 API
**Settings**: Follow n8n instructions to complete OAuth2 authorization flow

## 🔧 Slack App Configuration

### Event Subscriptions Settings
```yaml
Request URL: https://your-n8n-domain.com/webhook-test/slack-calendar
Subscribe to bot events:
  - message.channels
  - app_mention
```

### Bot Token Scopes
```yaml
Required permissions:
  - channels:read
  - channels:history
  - chat:write
  - users:read
  - app_mentions:read

Optional permissions:
  - chat:write.public
  - reactions:read
```

## 🚨 Known Issues and Solutions

### 1. Infinite Loop Issue
**Problem**: Bot-sent notifications trigger new workflows
**Solution**: Add Bot filtering in Filter Channel Messages
```javascript
Value 1: ={{$json.user}}
Operation: Not Equal
Value 2: YOUR_BOT_USER_ID
```

### 2. All Day Event Expression Issue
**Problem**: `={{$json.isAllDay ? 'Yes' : 'No'}}` not accepted
**Solution**: Use branch handling or fixed values
```javascript
// Solution 1: Branch handling
Event Type Filter → true → Create All Day Event (All Day: Yes)
                  → false → Create Timed Event (All Day: No)

// Solution 2: Try different expressions
={{Boolean($json.isAllDay)}}
={{$json.isAllDay === true}}
```

### 3. Confidence Judgment Issue
**Problem**: Float comparison failure
**Solution**: Use `parseFloat()` to ensure numeric type
```javascript
={{parseFloat($json.confidence) >= 0.7}}
```

### 4. All-day Event Time Format Issue
**Problem**: Google Calendar shows 08:00-08:00 instead of all-day
**Verification**: Check Google Calendar output format
```json
// ✅ Correct all-day event format
"start": {"date": "2025-05-20"}
"end": {"date": "2025-05-20"}

// ❌ Incorrect format
"start": {"dateTime": "2025-05-20T08:00:00+08:00"}
```

## 🧪 Test Cases

### All-day Event Test
```
Input: "5/20 go to Nagoya for fun"
Expected output:
- title: "Go to Nagoya for fun"
- startDateTime: "2025-05-20"
- endDateTime: "2025-05-20" 
- isAllDay: true
- location: "Nagoya"
- confidence: 0.9+
```

### Timed Event Test
```
Input: "Team meeting tomorrow 2 PM"
Expected output:
- title: "Team meeting"
- startDateTime: "2025-05-XX T14:00:00+08:00"
- endDateTime: "2025-05-XX T15:00:00+08:00"
- isAllDay: false
- confidence: 0.8+
```

### Low Confidence Test
```
Input: "Might have a meeting"
Expected result: Trigger Low Confidence Notification
```

### No Event Test
```
Input: "Weather is nice today"
Expected result: Trigger Error Notification (no_event)
```

## 🔍 Troubleshooting

### Debugging Steps
1. **Check Webhook trigger**: Confirm Slack events reach n8n
2. **Check filter conditions**: Confirm Filter Channel Messages logic is correct
3. **Check AI output**: View Basic LLM Chain raw response
4. **Check data processing**: View Process AI Response console logs
5. **Check confidence judgment**: Confirm Confidence Filter logic
6. **Check Google Calendar**: Confirm event creation success

### Common Debug Code
```javascript
// Add debugging in any Function node
console.log('Debug data:', JSON.stringify($json, null, 2));
console.log('Data type:', typeof $json.confidence);
console.log('Condition result:', $json.status === 'high_confidence');
return [$input.all()];
```

## 📈 Performance Optimization Suggestions

### 1. Reduce API Calls
- Optimize AI prompt to reduce redundant analysis
- Merge similar processing logic

### 2. Improve Error Handling
- Add retry mechanisms
- Improve user-friendly error messages

### 3. Enhance Features
- Support recurring events
- Add event editing functionality
- Integrate other calendar services

## 🆕 Extension Feature Suggestions

### 1. Meeting Room Booking Integration
- Check meeting room availability
- Auto-book meeting rooms

### 2. Participant Management
- Auto-invite relevant people
- Map Slack users to Gmail

### 3. Smart Suggestions
- Time suggestions based on historical data
- Conflict detection and alternative time suggestions

### 4. Multi-language Support
- Support English and other languages
- Internationalized time formats

## 📝 Maintenance Notes

### Regular Checks
- [ ] Slack Token validity
- [ ] Azure OpenAI Quota usage
- [ ] Google Calendar API limits
- [ ] Workflow execution logs

### Update Considerations
- [ ] AI Model version updates
- [ ] Slack API changes
- [ ] Google Calendar API changes
- [ ] n8n version compatibility

## 🔗 Related Documentation

- [n8n Official Documentation](https://docs.n8n.io/)
- [Slack API Documentation](https://api.slack.com/)
- [Google Calendar API](https://developers.google.com/calendar)
- [Azure OpenAI Documentation](https://docs.microsoft.com/en-us/azure/cognitive-services/openai/)

---

## 📧 Support Information

**System Version**: n8n 1.93.0
**Last Updated**: 2025-05-28
**Feature Status**: ✅ Production Ready

For issues, please provide:
1. Specific error messages
2. Relevant execution logs
3. Input test data
4. Expected output results