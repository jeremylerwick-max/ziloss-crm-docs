# FRONTEND STRUCTURE

**Estimated Lines:** 26,400
**Estimated Time:** 2-3 weeks
**Tech Stack:** React 18, TypeScript, Vite, Tailwind, Shadcn/ui

---

## PURPOSE

Complete frontend for the CRM including:
- Dashboard with analytics
- Conversation inbox (Intercom-style)
- Contact management
- Calendar/appointments
- Visual workflow builder
- Settings & integrations

---

## FILE STRUCTURE

```
/frontend/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.js
│
├── src/
│   ├── main.tsx               # Entry point
│   ├── App.tsx                # Root component
│   ├── routes.tsx             # Route definitions
│   │
│   ├── components/
│   │   ├── ui/                # Shadcn/Radix primitives (35 files)
│   │   │   ├── button.tsx
│   │   │   ├── input.tsx
│   │   │   ├── select.tsx
│   │   │   ├── modal.tsx
│   │   │   ├── table.tsx
│   │   │   ├── tabs.tsx
│   │   │   ├── dropdown.tsx
│   │   │   ├── toast.tsx
│   │   │   ├── badge.tsx
│   │   │   ├── avatar.tsx
│   │   │   ├── card.tsx
│   │   │   ├── checkbox.tsx
│   │   │   ├── radio.tsx
│   │   │   ├── switch.tsx
│   │   │   ├── slider.tsx
│   │   │   ├── progress.tsx
│   │   │   ├── skeleton.tsx
│   │   │   ├── tooltip.tsx
│   │   │   ├── popover.tsx
│   │   │   ├── command.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── sheet.tsx
│   │   │   ├── accordion.tsx
│   │   │   ├── alert.tsx
│   │   │   ├── calendar.tsx
│   │   │   ├── date-picker.tsx
│   │   │   ├── time-picker.tsx
│   │   │   ├── data-table.tsx
│   │   │   ├── pagination.tsx
│   │   │   ├── search.tsx
│   │   │   ├── empty-state.tsx
│   │   │   ├── loading.tsx
│   │   │   ├── error-boundary.tsx
│   │   │   └── index.ts
│   │   │
│   │   ├── layout/            # Layout components (4 files)
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Header.tsx
│   │   │   ├── MainLayout.tsx
│   │   │   └── AuthLayout.tsx
│   │   │
│   │   ├── contacts/          # Contact management (7 files)
│   │   │   ├── ContactTable.tsx
│   │   │   ├── ContactDetail.tsx
│   │   │   ├── ContactForm.tsx
│   │   │   ├── ContactFilters.tsx
│   │   │   ├── ContactImport.tsx
│   │   │   ├── ContactTimeline.tsx
│   │   │   └── ContactTags.tsx
│   │   │
│   │   ├── conversations/     # Inbox (6 files)
│   │   │   ├── ConversationList.tsx
│   │   │   ├── ConversationDetail.tsx
│   │   │   ├── MessageBubble.tsx
│   │   │   ├── MessageComposer.tsx
│   │   │   ├── ConversationFilters.tsx
│   │   │   └── QuickActions.tsx
│   │   │
│   │   ├── appointments/      # Calendar (7 files)
│   │   │   ├── Calendar.tsx
│   │   │   ├── CalendarDay.tsx
│   │   │   ├── CalendarWeek.tsx
│   │   │   ├── CalendarMonth.tsx
│   │   │   ├── AppointmentCard.tsx
│   │   │   ├── AppointmentForm.tsx
│   │   │   └── AvailabilitySettings.tsx
│   │   │
│   │   ├── workflows/         # Visual builder (25 files)
│   │   │   ├── WorkflowList.tsx
│   │   │   ├── WorkflowBuilder.tsx
│   │   │   ├── WorkflowCanvas.tsx
│   │   │   ├── WorkflowNode.tsx
│   │   │   ├── WorkflowEdge.tsx
│   │   │   ├── WorkflowSidebar.tsx
│   │   │   ├── WorkflowSettings.tsx
│   │   │   ├── WorkflowStats.tsx
│   │   │   ├── nodes/
│   │   │   │   ├── TriggerNode.tsx
│   │   │   │   ├── ActionNode.tsx
│   │   │   │   ├── ConditionNode.tsx
│   │   │   │   ├── WaitNode.tsx
│   │   │   │   ├── SMSNode.tsx
│   │   │   │   ├── EmailNode.tsx
│   │   │   │   ├── TagNode.tsx
│   │   │   │   ├── WebhookNode.tsx
│   │   │   │   └── EndNode.tsx
│   │   │   └── panels/
│   │   │       ├── TriggerPanel.tsx
│   │   │       ├── SMSPanel.tsx
│   │   │       ├── EmailPanel.tsx
│   │   │       ├── WaitPanel.tsx
│   │   │       ├── ConditionPanel.tsx
│   │   │       ├── TagPanel.tsx
│   │   │       └── WebhookPanel.tsx
│   │   │
│   │   ├── analytics/         # Dashboard (5 files)
│   │   │   ├── Dashboard.tsx
│   │   │   ├── MetricCard.tsx
│   │   │   ├── ConversationChart.tsx
│   │   │   ├── ResponseTimeChart.tsx
│   │   │   └── HeatmapChart.tsx
│   │   │
│   │   └── settings/          # Settings (5 files)
│   │       ├── GeneralSettings.tsx
│   │       ├── TeamSettings.tsx
│   │       ├── IntegrationSettings.tsx
│   │       ├── BillingSettings.tsx
│   │       └── APISettings.tsx
│   │
│   ├── pages/                 # Page components
│   │   ├── DashboardPage.tsx
│   │   ├── ConversationsPage.tsx
│   │   ├── ContactsPage.tsx
│   │   ├── CalendarPage.tsx
│   │   ├── WorkflowsPage.tsx
│   │   ├── SettingsPage.tsx
│   │   ├── LoginPage.tsx
│   │   └── NotFoundPage.tsx
│   │
│   ├── hooks/                 # Custom hooks (6 files)
│   │   ├── useContacts.ts
│   │   ├── useConversations.ts
│   │   ├── useAppointments.ts
│   │   ├── useWorkflows.ts
│   │   ├── useWebSocket.ts
│   │   └── useAuth.ts
│   │
│   ├── api/                   # API client (6 files)
│   │   ├── client.ts
│   │   ├── contacts.ts
│   │   ├── conversations.ts
│   │   ├── appointments.ts
│   │   ├── workflows.ts
│   │   └── auth.ts
│   │
│   ├── stores/                # Zustand stores (4 files)
│   │   ├── authStore.ts
│   │   ├── contactStore.ts
│   │   ├── conversationStore.ts
│   │   └── uiStore.ts
│   │
│   ├── types/                 # TypeScript types (5 files)
│   │   ├── contact.ts
│   │   ├── conversation.ts
│   │   ├── appointment.ts
│   │   ├── workflow.ts
│   │   └── api.ts
│   │
│   └── utils/                 # Utilities (4 files)
│       ├── date.ts
│       ├── format.ts
│       ├── validation.ts
│       └── constants.ts
│
└── public/
    └── assets/
```

---

## KEY COMPONENTS

### ConversationList.tsx (Inbox)
```tsx
interface ConversationListProps {
  conversations: Conversation[];
  selectedId?: string;
  onSelect: (id: string) => void;
  filters: ConversationFilters;
  onFilterChange: (filters: ConversationFilters) => void;
}

export function ConversationList({
  conversations,
  selectedId,
  onSelect,
  filters,
  onFilterChange
}: ConversationListProps) {
  return (
    <div className="flex flex-col h-full">
      {/* Filter tabs */}
      <div className="flex border-b p-2 gap-2">
        <Badge 
          variant={filters.status === 'all' ? 'default' : 'outline'}
          onClick={() => onFilterChange({ ...filters, status: 'all' })}
        >
          All
        </Badge>
        <Badge 
          variant={filters.status === 'unread' ? 'default' : 'outline'}
          onClick={() => onFilterChange({ ...filters, status: 'unread' })}
        >
          Unread ({counts.unread})
        </Badge>
        <Badge 
          variant={filters.status === 'starred' ? 'default' : 'outline'}
          onClick={() => onFilterChange({ ...filters, status: 'starred' })}
        >
          Starred
        </Badge>
      </div>
      
      {/* Conversation list */}
      <div className="flex-1 overflow-y-auto">
        {conversations.map(convo => (
          <ConversationRow
            key={convo.id}
            conversation={convo}
            isSelected={convo.id === selectedId}
            onClick={() => onSelect(convo.id)}
          />
        ))}
      </div>
    </div>
  );
}
```

### WorkflowCanvas.tsx (Visual Builder)
```tsx
import ReactFlow, { 
  Node, Edge, Controls, Background 
} from 'reactflow';

interface WorkflowCanvasProps {
  workflow: Workflow;
  onChange: (workflow: Workflow) => void;
  onNodeSelect: (node: WorkflowNode | null) => void;
}

export function WorkflowCanvas({
  workflow,
  onChange,
  onNodeSelect
}: WorkflowCanvasProps) {
  const nodes = workflowToNodes(workflow);
  const edges = workflowToEdges(workflow);
  
  const nodeTypes = {
    trigger: TriggerNode,
    action: ActionNode,
    condition: ConditionNode,
    wait: WaitNode,
  };
  
  const onNodesChange = useCallback((changes) => {
    const updated = applyNodeChanges(changes, nodes);
    onChange(nodesToWorkflow(updated, edges));
  }, [nodes, edges, onChange]);
  
  const onConnect = useCallback((connection) => {
    const updated = addEdge(connection, edges);
    onChange(nodesToWorkflow(nodes, updated));
  }, [nodes, edges, onChange]);
  
  return (
    <div className="h-full w-full">
      <ReactFlow
        nodes={nodes}
        edges={edges}
        nodeTypes={nodeTypes}
        onNodesChange={onNodesChange}
        onConnect={onConnect}
        onNodeClick={(_, node) => onNodeSelect(node.data)}
        fitView
      >
        <Background />
        <Controls />
      </ReactFlow>
    </div>
  );
}
```

---

## STATE MANAGEMENT

### conversationStore.ts (Zustand)
```ts
import { create } from 'zustand';
import { subscribeWithSelector } from 'zustand/middleware';

interface ConversationState {
  conversations: Conversation[];
  selectedId: string | null;
  filters: ConversationFilters;
  isLoading: boolean;
  
  // Actions
  setConversations: (conversations: Conversation[]) => void;
  selectConversation: (id: string | null) => void;
  setFilters: (filters: ConversationFilters) => void;
  addMessage: (conversationId: string, message: Message) => void;
  markAsRead: (conversationId: string) => void;
}

export const useConversationStore = create<ConversationState>()(
  subscribeWithSelector((set, get) => ({
    conversations: [],
    selectedId: null,
    filters: { status: 'all', assignee: null },
    isLoading: false,
    
    setConversations: (conversations) => set({ conversations }),
    
    selectConversation: (id) => set({ selectedId: id }),
    
    setFilters: (filters) => set({ filters }),
    
    addMessage: (conversationId, message) => set(state => ({
      conversations: state.conversations.map(c =>
        c.id === conversationId
          ? { ...c, messages: [...c.messages, message], lastMessageAt: message.createdAt }
          : c
      )
    })),
    
    markAsRead: (conversationId) => set(state => ({
      conversations: state.conversations.map(c =>
        c.id === conversationId
          ? { ...c, unreadCount: 0 }
          : c
      )
    })),
  }))
);
```

---

## REAL-TIME UPDATES

### useWebSocket.ts
```ts
import { useEffect } from 'react';
import { useConversationStore } from '@/stores/conversationStore';

export function useWebSocket() {
  const addMessage = useConversationStore(s => s.addMessage);
  
  useEffect(() => {
    const ws = new WebSocket(import.meta.env.VITE_WS_URL);
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      
      switch (data.type) {
        case 'message.new':
          addMessage(data.conversationId, data.message);
          // Play notification sound
          playNotificationSound();
          break;
          
        case 'conversation.updated':
          // Refresh conversation
          break;
      }
    };
    
    return () => ws.close();
  }, [addMessage]);
}
```

---

## LINE ESTIMATES

| Directory | Files | Lines |
|-----------|-------|-------|
| components/ui/ | 35 | 4,000 |
| components/layout/ | 4 | 800 |
| components/contacts/ | 7 | 2,500 |
| components/conversations/ | 6 | 3,000 |
| components/appointments/ | 7 | 2,500 |
| components/workflows/ | 25 | 8,000 |
| components/analytics/ | 5 | 1,500 |
| components/settings/ | 5 | 1,500 |
| pages/ | 8 | 800 |
| hooks/ | 6 | 800 |
| api/ | 6 | 600 |
| stores/ | 4 | 500 |
| types/ | 5 | 400 |
| utils/ | 4 | 300 |
| **TOTAL** | **119** | **26,400** |

---

## DEPENDENCIES

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "zustand": "^4.4.7",
    "@tanstack/react-query": "^5.8.4",
    "axios": "^1.6.2",
    "reactflow": "^11.10.1",
    "recharts": "^2.10.3",
    "date-fns": "^2.30.0",
    "zod": "^3.22.4",
    "@radix-ui/react-dialog": "^1.0.5",
    "@radix-ui/react-dropdown-menu": "^2.0.6",
    "@radix-ui/react-popover": "^1.0.7",
    "@radix-ui/react-select": "^2.0.0",
    "@radix-ui/react-tabs": "^1.0.4",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.0.0",
    "lucide-react": "^0.294.0",
    "tailwind-merge": "^2.1.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.37",
    "@types/react-dom": "^18.2.15",
    "@vitejs/plugin-react": "^4.2.0",
    "autoprefixer": "^10.4.16",
    "postcss": "^8.4.31",
    "tailwindcss": "^3.3.5",
    "typescript": "^5.2.2",
    "vite": "^5.0.0"
  }
}
```
