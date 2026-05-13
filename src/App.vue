<script setup lang="ts">
import { ref, computed, watch } from 'vue';
import { 
  DashboardOutlined, 
  FileTextOutlined, 
  TeamOutlined, 
  SettingOutlined, 
  PlusOutlined, 
  SearchOutlined,
  ArrowLeftOutlined,
  CloudServerOutlined,
  CheckCircleOutlined,
  CloseCircleOutlined,
  DownloadOutlined,
  InfoCircleOutlined,
  NodeIndexOutlined,
  UserOutlined,
  ClockCircleOutlined,
  FilterOutlined,
  ReloadOutlined,
  LinkOutlined,
  ClusterOutlined
} from '@ant-design/icons-vue';
import { message, Empty } from 'ant-design-vue';

// --- Types ---
interface Department {
  key: string;
  title: string;
  isCascading?: boolean;
  children?: Department[];
}

interface WorkOrder {
  id: string;
  name: string;
  priority: 'high' | 'medium' | 'low';
  template: string;
  templateCategory: 'alarm' | 'notification';
  type: string;
  createTime: string;
  duration: string;
  flowStatus: 'processing' | 'timeout' | 'completed' | 'rejected';
  responsibleUnits: string[];
  creator: string;
  updateTime: string;
  currentNode: string;
  isCascading: boolean;
  hasApprovalStage: boolean;
  hasFailure?: boolean;
  subPlatforms: {
    name: string;
    status: 'processing' | 'timeout' | 'approval' | 'completed' | 'rejected';
    timeConsumed: string;
    isLocal?: boolean;
    isProcessed?: boolean;
    feedback?: string;
    flowHistory: {
      node: string;
      status: string;
      responsible: string;
      time: string;
      content?: string;
      attachment?: {
        name: string;
        url: string;
      };
    }[];
  }[];
}

interface Template {
  id: string;
  title: string;
  category: 'alarm' | 'notification';
  type: string;
  description: string;
  hasApproval: boolean;
}

// --- Mock Data ---
const cascadingPlatforms = [
  '广东省气象局',
  '浙江省气象局',
  '四川省气象局',
  '福建省气象局',
  '西藏自治区气象局',
  '云南省气象局',
  '新疆维吾尔自治区气象局',
  '内蒙古自治区气象局'
];

const initialDepts: Department[] = [
  {
    key: 'all',
    title: '气象局组织架构',
    children: [
      { key: 'dept-1', title: '国家气象局安全室' },
      { key: 'dept-it', title: 'IT部' },
      { key: 'dept-info', title: '信息中心' },
      { key: 'dept-2', title: '广东省气象局', isCascading: true },
      { key: 'dept-3', title: '浙江省气象局', isCascading: true },
      { key: 'dept-4', title: '四川省气象局', isCascading: true },
      { key: 'dept-6', title: '西藏自治区气象局', isCascading: true },
      { key: 'dept-7', title: '福建省气象局', isCascading: true },
      { key: 'dept-10', title: '陕西省气象局', isCascading: true },
      { key: 'dept-14', title: '云南省气象局', isCascading: true },
      { key: 'dept-15', title: '新疆维吾尔自治区气象局', isCascading: true },
      { key: 'dept-16', title: '内蒙古自治区气象局', isCascading: true }
    ]
  }
];

const templates: Template[] = [
  {
    id: 'tpl-1',
    title: '资产专项盘点',
    category: 'alarm',
    type: '资产收集',
    description: '支持多个级联部门选择，包含处置与审批节点。适用于年度或专项国安资产台账收集。',
    hasApproval: true
  },
  {
    id: 'tpl-2',
    title: '资产退库申请',
    category: 'notification',
    type: '行政审批',
    description: '支持多个级联部门选择，仅包含处置节点。适用于下级单位申请核心资产报废、迁移等变更流程。',
    hasApproval: false
  },
  {
    id: 'tpl-3',
    title: '事件处置',
    category: 'alarm',
    type: '事件闭环',
    description: '仅允许跟事件关联的下级部门才可以选择。针对安全事件的闭环处置流程。',
    hasApproval: true
  },
  {
    id: 'tpl-4',
    title: '脆弱性处置',
    category: 'alarm',
    type: '漏洞闭环',
    description: '仅支持选择本级平台的部门，下级平台的部门不可勾选。用于本级资产的漏洞修复与核查。',
    hasApproval: true
  }
];

const allMembers = [
  { key: '1', name: '张处长', dept: 'dept-1', deptPath: '气象局组织架构/国家气象局安全室', role: '安全室主任' },
  { key: 'm-it', name: '李工', dept: 'dept-it', deptPath: '气象局组织架构/IT部', role: '系统管理员' },
  { key: 'm-info', name: '王工', dept: 'dept-info', deptPath: '气象局组织架构/信息中心', role: '网络工程师' },
  { key: '2', name: '广东局管理员', dept: 'dept-2', deptPath: '气象局组织架构/广东省气象局', role: '信息中心主管' },
  { key: '3', name: '浙江局管理员', dept: 'dept-3', deptPath: '气象局组织架构/浙江省气象局', role: '安全员' },
  { key: '4', name: '西藏局联络人', dept: 'dept-6', deptPath: '气象局组织架构/西藏自治区气象局', role: '综合办' },
  { key: '5', name: '四川局管理员', dept: 'dept-4', deptPath: '气象局组织架构/四川省气象局', role: '技术保障' },
];

const initialOrders: WorkOrder[] = [
  {
    id: 'MET-2024-001',
    name: '2024年度国安资产专项盘点',
    priority: 'high',
    template: '资产专项盘点',
    templateCategory: 'alarm',
    type: '级联下发',
    createTime: '2024-04-13 09:00:00',
    duration: '4小时',
    flowStatus: 'timeout',
    responsibleUnits: ['广东省气象局', '西藏自治区气象局', '四川省气象局', '浙江省气象局'],
    creator: '张处长',
    updateTime: '2024-04-13 13:00:00',
    currentNode: '资产填报',
    isCascading: true,
    hasApprovalStage: true,
    hasFailure: true,
    subPlatforms: [
      {
        name: '广东省气象局',
        status: 'approval',
        timeConsumed: '2小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-13 09:00:00' },
          { 
            node: '录入信息', 
            status: '已提交资产表', 
            responsible: '广东局管理员', 
            time: '2024-04-13 11:00:00',
            content: '已完成本季度资产盘点，详见附件。',
            attachment: { name: '2024年度资产盘点表_广东局.xlsx', url: '#' }
          },
          { node: '等待审核', status: '待审批', responsible: '张处长', time: '2024-04-13 11:05:00' }
        ]
      },
      {
        name: '四川省气象局',
        status: 'processing',
        timeConsumed: '4小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-13 09:00:00' },
          { node: '审核', status: '已驳回', responsible: '张处长', time: '2024-04-13 10:30:00' }
        ]
      },
      {
        name: '西藏自治区气象局',
        status: 'timeout',
        timeConsumed: '8小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-13 09:00:00' },
          { node: '催办', status: '已催办', responsible: '张处长', time: '2024-04-13 15:00:00', content: '年度盘点即将截止，请西藏局加快进度。' }
        ]
      },
      {
        name: '福建省气象局',
        status: 'approval',
        timeConsumed: '5小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-13 09:00:00' },
          { node: '录入信息', status: '已提交资产表', responsible: '福建局管理员', time: '2024-04-13 14:00:00' }
        ]
      },
      {
        name: '浙江省气象局',
        status: 'completed',
        timeConsumed: '1.5小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-13 09:00:00' },
          { node: '录入信息', status: '已提交资产表', responsible: '浙江局管理员', time: '2024-04-13 10:20:00' },
          { node: '审核', status: '审批通过', responsible: '张处长', time: '2024-04-13 10:30:00' }
        ]
      }
    ]
  },
  {
    id: 'MET-2024-002',
    name: '主机漏洞紧急修复任务',
    priority: 'high',
    template: '事件处置',
    templateCategory: 'alarm',
    type: '级联下发',
    createTime: '2024-04-14 10:00:00',
    duration: '24小时',
    flowStatus: 'timeout',
    responsibleUnits: ['国家气象局安全室', '陕西省气象局', '云南省气象局'],
    creator: '张处长',
    updateTime: '2024-04-14 10:00:00',
    currentNode: '处置',
    isCascading: true,
    hasApprovalStage: true,
    subPlatforms: [
      {
        name: '国家气象局安全室',
        status: 'approval',
        timeConsumed: '24小时',
        isLocal: true,
        isProcessed: true,
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-14 10:00:00' },
          { node: '处置', status: '已提交', responsible: '张处长', time: '2024-04-14 15:00:00', content: '已完成漏洞补丁更新。' }
        ]
      },
      {
        name: '陕西省气象局',
        status: 'timeout',
        timeConsumed: '24小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-14 10:00:00' }
        ]
      },
      {
        name: '云南省气象局',
        status: 'processing',
        timeConsumed: '24小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-14 10:00:00' }
        ]
      },
      {
        name: '新疆维吾尔自治区气象局',
        status: 'processing',
        timeConsumed: '24小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-14 10:00:00' }
        ]
      }
    ]
  },
  {
    id: 'MET-2024-004',
    name: '资产退库申请流程',
    priority: 'low',
    template: '资产退库申请',
    templateCategory: 'notification',
    type: '级联下发',
    createTime: '2024-04-15 09:00:00',
    duration: '2小时',
    flowStatus: 'processing',
    responsibleUnits: ['浙江省气象局', '福建省气象局'],
    creator: '张处长',
    updateTime: '2024-04-15 09:00:00',
    currentNode: '处置',
    isCascading: true,
    hasApprovalStage: false,
    subPlatforms: [
      {
        name: '浙江省气象局',
        status: 'processing',
        timeConsumed: '2小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-15 09:00:00' }
        ]
      },
      {
        name: '福建省气象局',
        status: 'processing',
        timeConsumed: '2小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-15 09:00:00' }
        ]
      }
    ]
  },
  {
    id: 'MET-2024-003',
    name: '全网安全设备巡检通知',
    priority: 'medium',
    template: '资产专项盘点',
    templateCategory: 'alarm',
    type: '级联下发',
    createTime: '2024-04-14 11:00:00',
    duration: '1小时',
    flowStatus: 'processing',
    hasFailure: true,
    responsibleUnits: ['湖南省气象局', '云南省气象局', '新疆维吾尔自治区气象局', '内蒙古自治区气象局'],
    creator: '张处长',
    updateTime: '2024-04-14 11:00:00',
    currentNode: '处置',
    isCascading: true,
    hasApprovalStage: true,
    subPlatforms: [
      {
        name: '湖南省气象局',
        status: 'processing',
        timeConsumed: '1小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-14 11:00:00' }
        ]
      },
      {
        name: '云南省气象局',
        status: 'processing',
        timeConsumed: '1小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-14 11:00:00' }
        ]
      },
      {
        name: '新疆维吾尔自治区气象局',
        status: 'processing',
        timeConsumed: '1小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-14 11:00:00' }
        ]
      },
      {
        name: '内蒙古自治区气象局',
        status: 'processing',
        timeConsumed: '1小时',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-14 11:00:00' }
        ]
      }
    ]
  },
  {
    id: 'MET-2024-005',
    name: '脆弱性扫描发现高危漏洞修复',
    priority: 'high',
    template: '脆弱性处置',
    templateCategory: 'alarm',
    type: '本级下发',
    createTime: '2024-04-15 14:00:00',
    duration: '2小时',
    flowStatus: 'processing',
    responsibleUnits: ['IT部'],
    creator: '张处长',
    updateTime: '2024-04-15 14:00:00',
    currentNode: '处置',
    isCascading: false,
    hasApprovalStage: true,
    subPlatforms: [
      {
        name: 'IT部',
        status: 'processing',
        timeConsumed: '2小时',
        isLocal: true,
        isProcessed: false,
        flowHistory: [{ node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-15 14:00:00' }]
      }
    ]
  },
  {
    id: 'MET-2024-006',
    name: '全网恶意IP紧急封禁指令',
    priority: 'high',
    template: '事件处置',
    templateCategory: 'alarm',
    type: '级联下发',
    createTime: '2024-04-15 17:00:00',
    duration: '30分钟',
    flowStatus: 'processing',
    responsibleUnits: ['浙江省气象局', '四川省气象局', '福建省气象局'],
    creator: '张处长',
    updateTime: '2024-04-15 17:00:00',
    currentNode: '处置',
    isCascading: true,
    hasApprovalStage: true,
    subPlatforms: [
      {
        name: '浙江省气象局',
        status: 'processing',
        timeConsumed: '30分钟',
        flowHistory: [{ node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-15 17:00:00' }]
      },
      {
        name: '四川省气象局',
        status: 'processing',
        timeConsumed: '30分钟',
        flowHistory: [{ node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-15 17:00:00' }]
      },
      {
        name: '福建省气象局',
        status: 'processing',
        timeConsumed: '30分钟',
        flowHistory: [{ node: '发起工单', status: '已发起', responsible: '张处长', time: '2024-04-15 17:00:00' }]
      }
    ]
  }
];

// --- State ---
const currentView = ref<'list' | 'template' | 'create' | 'detail' | 'dept'>('list');
const workOrders = ref<WorkOrder[]>(initialOrders);
const departments = ref<Department[]>(initialDepts);
const selectedTemplate = ref<Template | null>(null);
const currentTitle = computed(() => {
  switch (currentView.value) {
    case 'list': return '全部工单';
    case 'template': return '发起流程';
    case 'create': return '发起工单';
    case 'detail': return '工单详情';
    case 'dept': return '人员管理';
    default: return '工单管理';
  }
});

const selectedOrder = ref<WorkOrder | null>(null);

// Modals
const isAddDeptModalVisible = ref(false);
const isEditDeptModalVisible = ref(false);
const isAddMemberModalVisible = ref(false);
const isResponsibleModalVisible = ref(false);
const isEventModalVisible = ref(false);
const isProcessDrawerVisible = ref(false);

const processForm = ref({
  result: 'close',
  desc: '',
  cc: [] as string[]
});

const editingDept = ref<Department | null>(null);
const isRetrying = ref(false);
const handleRetry = () => {
  isRetrying.value = true;
  setTimeout(() => {
    isRetrying.value = false;
    if (selectedOrder.value) {
      selectedOrder.value.hasFailure = false;
    }
    message.success('重新尝试成功，数据已同步');
  }, 1500);
};
const expandedStages = ref<string[]>([]);

const toggleStage = (stage: string) => {
  if (expandedStages.value.includes(stage)) {
    expandedStages.value = expandedStages.value.filter(s => s !== stage);
  } else {
    expandedStages.value.push(stage);
  }
};

watch(selectedOrder, (newVal) => {
  if (newVal) {
    const stages = [];
    // Only expand if there are items in that stage
    const hasDisposal = newVal.subPlatforms.some(s => s.status !== 'approval' && s.status !== 'completed');
    const hasApproval = newVal.subPlatforms.some(s => s.status === 'approval' || s.status === 'completed');
    
    if (hasDisposal) stages.push('disposal');
    if (hasApproval) stages.push('approval');
    
    expandedStages.value = stages;
  }
}, { immediate: true });

const subPlatformStats = computed(() => {
  if (!selectedOrder.value) return { total: 0, local: 0, cascading: 0 };
  const total = selectedOrder.value.subPlatforms.length;
  const local = selectedOrder.value.subPlatforms.filter(s => s.isLocal).length;
  return { total, local, cascading: total - local };
});

const eventList = ref([
  { id: 'ev-1', status: '待处置', time: '2024-04-14 16:06:00', name: '主机存在RDP远程桌面漏洞', asset: '10.5.40.20', platform: '本平台', level: '中危' },
  { id: 'ev-2', status: '待处置', time: '2024-04-14 10:12:00', name: '主机存在疑似Web应用攻击', asset: '192.168.201.223', platform: '本平台', level: '严重' },
  { id: 'ev-3', status: '待处置', time: '2024-04-14 10:12:00', name: '主机存在疑似执行病毒', asset: '10.122.230.7', platform: '本平台', level: '高危' },
  { id: 'ev-4', status: '待处置', time: '2024-04-14 10:00:00', name: '主机存在通过Certutil下载恶意文件', asset: '192.168.115.49', platform: '本平台', level: '高危' },
  { id: 'ev-5', status: '待处置', time: '2024-04-14 10:00:00', name: '主机存在RCE利用、反弹Shell', asset: '172.23.162.163', platform: '级联下级平台', level: '低危', subPlatform: '广东省气象局' },
  { id: 'ev-6', status: '待处置', time: '2024-04-14 10:00:00', name: '主机存在疑似Web应用攻击', asset: '172.27.94.70', platform: '级联下级平台', level: '严重', subPlatform: '浙江省气象局' },
]);

const selectedEvents = ref<any[]>([]);
const eventPlatformFilter = ref('all');

const filteredEventList = computed(() => {
  if (eventPlatformFilter.value === 'all') return eventList.value;
  return eventList.value.filter(e => e.platform === eventPlatformFilter.value);
});

const tempSelectedEvents = ref<any[]>([]);

const selectedDeptKey = ref<string>('all');
const cleanDepartments = computed(() => {
  const mapNodes = (nodes: Department[]): any[] => {
    // 排序：普通部门在前，级联部门在后
    const sortedNodes = [...nodes].sort((a, b) => {
      if (a.isCascading === b.isCascading) return 0;
      return a.isCascading ? 1 : -1;
    });

    return sortedNodes.map(n => ({
      key: n.key,
      title: n.title,
      isCascading: n.isCascading,
      children: n.children ? mapNodes(n.children) : undefined
    }));
  };
  return mapNodes(departments.value);
});

const isCascadingSelected = computed(() => {
  const findNode = (nodes: Department[], key: string): Department | null => {
    for (const node of nodes) {
      if (node.key === key) return node;
      if (node.children) {
        const found = findNode(node.children, key);
        if (found) return found;
      }
    }
    return null;
  };
  const selected = findNode(departments.value, selectedDeptKey.value);
  return selected?.isCascading || false;
});

const handleRefreshMembers = (deptKey: string) => {
  // 移除刷新逻辑，但保留函数定义以防其他地方引用（虽然应该没有了）
};

const filteredMembers = computed(() => {
  if (selectedDeptKey.value === 'all') return allMembers;
  return allMembers.filter(m => m.dept === selectedDeptKey.value);
});

// Forms
const orderForm = ref({
  name: '',
  priority: 'medium',
  responsibleUnits: [] as string[]
});

const deptForm = ref({
  parent: 'all',
  name: '',
  cascadingPlatform: undefined as string | undefined
});

const treeDataWithDisabled = computed(() => {
  const isEventHandling = selectedTemplate.value?.title === '事件处置';
  const isVulnerabilityDisposal = selectedTemplate.value?.title === '脆弱性处置';
  const cascadingEvent = selectedEvents.value.find(e => e.platform === '级联下级平台');
  const allowedSubPlatform = cascadingEvent?.subPlatform;

  const mapNodes = (nodes: any[]): any[] => nodes.map(n => {
    let disabled = false;
    let tooltip = '';
    
    if (isEventHandling && cascadingEvent) {
      // Local depts are always allowed: 国家气象局安全室, IT部, 信息中心
      const isLocal = ['国家气象局安全室', 'IT部', '信息中心'].includes(n.title);
      const isTargetSub = n.title === allowedSubPlatform;
      if (n.isCascading && !isTargetSub) {
        disabled = true;
        tooltip = '仅允许上报该事件的级联平台进行处置';
      }
    }

    if (isVulnerabilityDisposal && n.isCascading) {
      disabled = true;
      tooltip = '该模板仅支持本级平台部门处置';
    }

    const newNode: any = {
      key: n.key,
      title: n.title,
      isCascading: n.isCascading,
      disableCheckbox: disabled,
      tooltip
    };
    if (n.children) {
      newNode.children = mapNodes(n.children);
    }
    return newNode;
  });
  
  return mapNodes(departments.value);
});

const tempResponsible = ref<string[]>([]);

// Drawer for sub-platform flow details
const isFlowDrawerVisible = ref(false);
const currentSubPlatform = ref<any>(null);
const currentOrderIndex = ref(0);

// --- Actions ---
const handleDeptSelect = (keys: any) => {
  selectedDeptKey.value = (keys[0] as string) || 'all';
};

const handleAddDept = () => {
  const newDept: Department = {
    key: `dept-${Date.now()}`,
    title: deptForm.value.name,
    isCascading: !!deptForm.value.cascadingPlatform
  };
  
  const addNode = (nodes: Department[]) => {
    for (const node of nodes) {
      if (node.key === deptForm.value.parent) {
        if (!node.children) node.children = [];
        node.children.push(newDept);
        return true;
      }
      if (node.children && addNode(node.children)) return true;
    }
    return false;
  };

  if (deptForm.value.parent === 'all') {
    if (!departments.value[0].children) departments.value[0].children = [];
    departments.value[0].children.push(newDept);
  } else {
    addNode(departments.value);
  }
  
  isAddDeptModalVisible.value = false;
  deptForm.value = { parent: 'all', name: '', cascadingPlatform: undefined };
  message.success('部门添加成功');
};

const handleEditDept = (node: any) => {
  editingDept.value = node;
  deptForm.value = {
    parent: 'all', // Simplified for demo
    name: node.title,
    cascadingPlatform: undefined
  };
  isEditDeptModalVisible.value = true;
};

const saveEditDept = () => {
  if (!editingDept.value) return;
  const updateNode = (nodes: Department[]) => {
    for (const node of nodes) {
      if (node.key === editingDept.value?.key) {
        node.title = deptForm.value.name;
        return true;
      }
      if (node.children && updateNode(node.children)) return true;
    }
    return false;
  };
  updateNode(departments.value);
  isEditDeptModalVisible.value = false;
  message.success('部门修改成功');
};

const handleDeleteDept = (key: string) => {
  const deleteNode = (nodes: Department[]) => {
    for (let i = 0; i < nodes.length; i++) {
      if (nodes[i].key === key) {
        nodes.splice(i, 1);
        return true;
      }
      if (nodes[i].children && deleteNode(nodes[i].children)) return true;
    }
    return false;
  };
  deleteNode(departments.value);
  message.success('部门删除成功');
};

const handleCreateOrder = () => {
  if (!selectedTemplate.value) return;
  
  const newOrder: WorkOrder = {
    id: `MET-${Date.now().toString().slice(-6)}`,
    name: orderForm.value.name || (selectedEvents.value.length > 0 ? selectedEvents.value[0].name : selectedTemplate.value.title),
    priority: orderForm.value.priority as any,
    template: selectedTemplate.value.title,
    templateCategory: selectedTemplate.value.category,
    type: selectedTemplate.value.hasApproval ? '级联下发' : '普通下发',
    createTime: new Date().toLocaleString(),
    duration: '0分',
    flowStatus: 'processing',
    responsibleUnits: orderForm.value.responsibleUnits,
    creator: '张处长',
    updateTime: new Date().toLocaleString(),
    currentNode: '处置',
    isCascading: orderForm.value.responsibleUnits.some(u => 
      departments.value[0].children?.find(d => d.title === u)?.isCascading
    ),
    hasApprovalStage: selectedTemplate.value.hasApproval,
    subPlatforms: orderForm.value.responsibleUnits
      .filter(u => departments.value[0].children?.find(d => d.title === u)?.isCascading)
      .map(u => ({ 
        name: u, 
        status: 'processing',
        timeConsumed: '0分',
        flowHistory: [
          { node: '发起工单', status: '已发起', responsible: '张处长', time: new Date().toLocaleString() }
        ]
      }))
  };

  workOrders.value.unshift(newOrder);
  currentView.value = 'list';
  orderForm.value = { name: '', priority: 'medium', responsibleUnits: [] };
  selectedEvents.value = [];
  message.success('工单发起成功');
};

const handleApprove = (orderId: string, unitName: string, action: 'pass' | 'reject') => {
  const order = workOrders.value.find(o => o.id === orderId);
  if (order) {
    const sub = order.subPlatforms.find(s => s.name === unitName);
    if (sub) {
      if (action === 'pass') {
        sub.status = 'completed';
        sub.flowHistory.push({
          node: '审核',
          status: '审批通过',
          responsible: '张处长',
          time: new Date().toLocaleString()
        });
      } else {
        sub.status = 'processing'; // Return to disposal
        sub.flowHistory.push({
          node: '审核',
          status: '已驳回',
          responsible: '张处长',
          time: new Date().toLocaleString()
        });
      }
    }
    const allCompleted = order.subPlatforms.every(s => s.status === 'completed');
    if (allCompleted) {
      order.flowStatus = 'completed';
    }
  }
  message.success(action === 'pass' ? '审批通过' : '已驳回');
};

const handleUrge = (sub: any) => {
  if (sub) {
    sub.flowHistory.push({
      node: '催办',
      status: '已催办',
      responsible: '张处长',
      time: new Date().toLocaleString(),
      content: '年度资产盘点即将截止，请加快进度。'
    });
  }
  message.success('催办成功');
};

const showFlowDrawer = (sub: any, index: number) => {
  currentSubPlatform.value = sub;
  currentOrderIndex.value = index;
  isFlowDrawerVisible.value = true;
};

const handlePrevNext = (direction: 'prev' | 'next') => {
  if (!selectedOrder.value) return;
  const subs = selectedOrder.value.subPlatforms;
  let newIndex = direction === 'prev' ? currentOrderIndex.value - 1 : currentOrderIndex.value + 1;
  if (newIndex >= 0 && newIndex < subs.length) {
    currentOrderIndex.value = newIndex;
    currentSubPlatform.value = subs[newIndex];
    approvalResult.value = 'pass'; // Reset
  }
};

const approvalResult = ref<'pass' | 'reject'>('pass');

const simulateSubPlatformComplete = (sub: any) => {
  if (!selectedOrder.value) return;
  if (selectedOrder.value.hasApprovalStage) {
    sub.status = 'approval';
    sub.flowHistory.push({
      node: '录入信息',
      status: '已提交附件',
      responsible: '下级平台',
      time: new Date().toLocaleString()
    });
    message.info(`${sub.name} 已完成处置，进入审核阶段`);
  } else {
    sub.status = 'completed';
    sub.flowHistory.push({
      node: '录入信息',
      status: '已完成',
      responsible: '下级平台',
      time: new Date().toLocaleString()
    });
    message.success(`${sub.name} 已完成处置`);
    
    // Check overall completion
    const allDone = selectedOrder.value.subPlatforms.every(s => s.status === 'completed');
    if (allDone) {
      selectedOrder.value.flowStatus = 'completed';
      selectedOrder.value.currentNode = '结束';
    }
  }
};

const submitApproval = () => {
  if (!selectedOrder.value || !currentSubPlatform.value) return;
  handleApprove(selectedOrder.value.id, currentSubPlatform.value.name, approvalResult.value);
  isFlowDrawerVisible.value = false;
};

const disposalFilter = ref('all');
const approvalFilter = ref('all');

const filteredSubPlatforms = computed(() => {
  if (!selectedOrder.value) return [];
  
  // For orders with approval, disposal only shows processing/timeout
  // For orders without approval, disposal shows processing/timeout/completed
  let subs = [];
  if (selectedOrder.value.hasApprovalStage) {
    subs = selectedOrder.value.subPlatforms.filter(s => s.status === 'processing' || s.status === 'timeout');
  } else {
    subs = selectedOrder.value.subPlatforms.filter(s => s.status === 'processing' || s.status === 'timeout' || s.status === 'completed');
  }

  if (disposalFilter.value === 'all') return subs;
  return subs.filter(s => s.status === disposalFilter.value);
});

const approvalSubPlatforms = computed(() => {
  if (!selectedOrder.value) return [];
  // Approval stage shows 'approval' (pending) and 'completed' (done)
  const subs = selectedOrder.value.subPlatforms.filter(s => s.status === 'approval' || s.status === 'completed');
  
  if (approvalFilter.value === 'all') return subs;
  if (approvalFilter.value === 'approval') return subs.filter(s => s.status === 'approval');
  if (approvalFilter.value === 'completed') return subs.filter(s => s.status === 'completed');
  return subs;
});

const getDisposalCounts = computed(() => {
  if (!selectedOrder.value) return { all: 0, processing: 0, timeout: 0, completed: 0 };
  const subs = selectedOrder.value.subPlatforms;
  return {
    all: selectedOrder.value.hasApprovalStage 
      ? subs.filter(s => s.status === 'processing' || s.status === 'timeout').length
      : subs.filter(s => s.status === 'processing' || s.status === 'timeout' || s.status === 'completed').length,
    processing: subs.filter(s => s.status === 'processing').length,
    timeout: subs.filter(s => s.status === 'timeout').length,
    completed: subs.filter(s => s.status === 'completed').length
  };
});

const getApprovalCounts = computed(() => {
  if (!selectedOrder.value) return { all: 0, approval: 0, completed: 0 };
  const subs = selectedOrder.value.subPlatforms;
  return {
    all: subs.filter(s => s.status === 'approval' || s.status === 'completed').length,
    approval: subs.filter(s => s.status === 'approval').length,
    completed: subs.filter(s => s.status === 'completed').length
  };
});

const findKeyByTitle = (nodes: Department[], title: string): string | undefined => {
  for (const n of nodes) {
    if (n.title === title) return n.key;
    if (n.children) {
      const k = findKeyByTitle(n.children, title);
      if (k) return k;
    }
  }
};

const onCheck = (checkedKeys: any, info: any) => {
  const titles = info.checkedNodes.map((n: any) => n.title).filter((t: string) => t !== '全部');
  tempResponsible.value = titles;
};

const handleProcessSubmit = () => {
  if (!currentSubPlatform.value || !selectedOrder.value) return;
  
  currentSubPlatform.value.isProcessed = true;
  currentSubPlatform.value.status = 'approval';
  
  const historyItem: any = {
    node: '处置',
    status: '已提交',
    responsible: '张处长',
    time: new Date().toLocaleString(),
    content: processForm.value.desc
  };

  // Simulate attachment for Asset Inventory
  if (selectedOrder.value.template === '资产专项盘点') {
    historyItem.attachment = { name: '资产盘点附件_已上传.pdf', url: '#' };
  }

  currentSubPlatform.value.flowHistory.push(historyItem);
  
  isProcessDrawerVisible.value = false;
  message.success('工单处理成功，已进入审核阶段');
};

</script>

<template>
  <a-config-provider
    :theme="{
      token: {
        colorPrimary: '#0052D9',
        borderRadius: 4,
        fontFamily: 'Inter, system-ui, sans-serif',
      },
    }"
  >
    <div class="w-[1920px] h-[1080px] bg-gray-100 flex items-center justify-center overflow-hidden mx-auto shadow-2xl">
      <a-layout class="w-full h-full">
        <a-layout-sider :width="240" style="background: #1d2129">
          <div class="p-4 flex items-center gap-3 mb-4">
            <div class="w-8 h-8 bg-[#0052D9] rounded flex items-center justify-center text-white font-bold">X</div>
            <div class="text-white">
              <div class="font-bold leading-none">Sangfor XDR</div>
              <div class="text-[10px] opacity-50 mt-1">Powered by Vue</div>
            </div>
          </div>
          <a-menu
            theme="dark"
            mode="inline"
            style="background: transparent"
            :selected-keys="[currentView === 'dept' ? 'dept' : 'order']"
          >
            <a-menu-item key="dash">
              <template #icon><DashboardOutlined /></template>
              工单总览
            </a-menu-item>
            <a-menu-item key="create-nav" @click="currentView = 'template'">
              <template #icon><PlusOutlined /></template>
              发起流程
            </a-menu-item>
            <a-sub-menu key="order">
              <template #icon><FileTextOutlined /></template>
              <template #title>工单管理</template>
              <a-menu-item key="all-order" @click="currentView = 'list'">全部工单</a-menu-item>
              <a-menu-item key="my-order">我的工单</a-menu-item>
              <a-menu-item key="archive">归档工单</a-menu-item>
            </a-sub-menu>
            <a-menu-item key="tpl-nav">
              <template #icon><NodeIndexOutlined /></template>
              流程模板
            </a-menu-item>
            <a-menu-item key="dept" @click="currentView = 'dept'">
              <template #icon><TeamOutlined /></template>
              人员管理
            </a-menu-item>
            <a-menu-item key="settings">
              <template #icon><SettingOutlined /></template>
              设置
            </a-menu-item>
          </a-menu>
        </a-layout-sider>

        <a-layout class="flex flex-col h-full">
          <a-layout-header class="bg-white px-6 flex items-center justify-between border-b border-[#f0f0f0] z-20 h-14 shrink-0 shadow-sm">
            <div class="flex items-center gap-4">
              <template v-if="currentView === 'detail' || currentView === 'create'">
                <a-button type="text" @click="currentView = currentView === 'detail' ? 'list' : 'template'" class="p-0 flex items-center justify-center">
                  <template #icon><ArrowLeftOutlined class="text-black" /></template>
                </a-button>
                <div class="text-base font-medium text-black">{{ currentView === 'detail' ? '工单详情' : '发起流程' }}</div>
              </template>
              <div v-else class="text-base font-medium text-black">
                {{ currentTitle }}
              </div>
            </div>
            <a-space size="large">
              <a-badge :count="99" size="small">
                <a-button type="text">
                  <template #icon><InfoCircleOutlined class="text-black" /></template>
                </a-button>
              </a-badge>
              <div class="flex items-center gap-2 cursor-pointer text-black">
                <div class="w-8 h-8 bg-blue-100 text-blue-600 rounded-full flex items-center justify-center font-bold">张</div>
                <span>张处长</span>
              </div>
            </a-space>
          </a-layout-header>

          <a-layout-content class="flex-1 overflow-hidden relative bg-[#f0f2f5]">
            <div class="absolute inset-0 overflow-y-auto p-4">
              
              <!-- List View -->
              <div v-if="currentView === 'list'" class="h-full flex flex-col space-y-4">
                <div class="bg-white p-4 rounded-sm shadow-sm">
                  <div class="flex justify-between items-center">
                    <a-space size="middle">
                      <a-input-search placeholder="输入工单名称搜索" style="width: 300px" />
                      <a-select defaultValue="all" style="width: 120px">
                        <a-select-option value="all">全部状态</a-select-option>
                      </a-select>
                    </a-space>
                    <a-button type="primary" @click="currentView = 'template'">
                      <template #icon><PlusOutlined /></template>
                      发起工单
                    </a-button>
                  </div>
                </div>

                <div class="flex-1 bg-white p-4 rounded-sm shadow-sm overflow-hidden flex flex-col">
                  <div class="mb-4 flex gap-2">
                    <a-button disabled>批量归档</a-button>
                    <a-button><template #icon><DownloadOutlined /></template>导出</a-button>
                  </div>
                  <div class="flex-1 overflow-auto">
                    <a-table
                      :dataSource="workOrders"
                      rowKey="id"
                      size="small"
                      :pagination="{ total: 887, showSizeChanger: true, showQuickJumper: true }"
                    >
                      <a-table-column title="序号" key="index">
                        <template #default="{ index }">{{ index + 1 }}</template>
                      </a-table-column>
                      <a-table-column title="工单名称" dataIndex="name">
                        <template #default="{ text, record }">
                          <div class="flex items-center gap-2">
                            <a-button type="link" class="p-0" @click="selectedOrder = record; currentView = 'detail'">{{ text }}</a-button>
                          </div>
                        </template>
                      </a-table-column>
                      <a-table-column title="紧急程度" dataIndex="priority">
                        <template #default="{ text }">
                          <a-tag :color="text === 'high' ? 'red' : text === 'medium' ? 'orange' : 'blue'">
                            {{ text === 'high' ? '高' : text === 'medium' ? '中' : '低' }}
                          </a-tag>
                        </template>
                      </a-table-column>
                      <a-table-column title="匹配模板" dataIndex="template" />
                      <a-table-column title="流程类型" dataIndex="type" />
                      <a-table-column title="创建时间" dataIndex="createTime" />
                      <a-table-column title="流程用时" dataIndex="duration">
                        <template #default="{ text, record }">
                          <div class="flex items-center gap-1">
                            <ClockCircleOutlined v-if="record.flowStatus === 'timeout'" class="text-red-500" />
                            <span>{{ text }}</span>
                          </div>
                        </template>
                      </a-table-column>
                      <a-table-column title="流转状态" dataIndex="flowStatus">
                        <template #default="{ text, record }">
                          <div class="flex items-center gap-1">
                            <span :class="text === 'completed' ? 'text-green-500' : text === 'timeout' ? 'text-red-500' : 'text-blue-500'">
                              {{ (text === 'processing' || text === 'timeout') ? '流转中' : '已结束' }}
                            </span>
                            <a-tooltip v-if="record.hasFailure" title="当前存在部分发送失败的工单">
                              <InfoCircleOutlined class="text-yellow-500 cursor-help" />
                            </a-tooltip>
                          </div>
                        </template>
                      </a-table-column>
                      <a-table-column title="当前责任人" dataIndex="responsibleUnits">
                        <template #default="{ text, record }">
                          <div>
                            {{ text.join(', ') }}
                            <div v-if="record.isCascading" class="text-[10px] text-gray-400">关联下级：{{ record.subPlatforms.length }}个单位</div>
                          </div>
                        </template>
                      </a-table-column>
                      <a-table-column title="发起人" dataIndex="creator" />
                      <a-table-column title="操作" key="action">
                        <template #default>
                          <a-button type="link" size="small">关闭工单</a-button>
                        </template>
                      </a-table-column>
                    </a-table>
                  </div>
                </div>
              </div>

              <!-- Template Selection -->
              <div v-else-if="currentView === 'template'" class="h-full flex flex-col space-y-4">
                <div class="bg-white p-4 rounded-sm shadow-sm flex items-center gap-4">
                  <a-button type="text" @click="currentView = 'list'"><template #icon><ArrowLeftOutlined /></template>返回</a-button>
                  <span class="font-medium text-base">选择模板</span>
                </div>
                <div class="bg-white p-6 rounded-sm shadow-sm flex-1 overflow-y-auto">
                  <div class="mb-6 flex justify-between items-center">
                    <a-space>
                      <a-select defaultValue="all" style="width: 120px">
                        <a-select-option value="all">全部分类</a-select-option>
                      </a-select>
                      <a-input placeholder="请输入模板名称" style="width: 300px">
                        <template #prefix><SearchOutlined /></template>
                      </a-input>
                    </a-space>
                    <a-button><template #icon><ReloadOutlined /></template></a-button>
                  </div>
                  <a-row :gutter="[16, 16]">
                    <a-col :span="24" v-for="tpl in templates" :key="tpl.id">
                      <a-card size="small" class="hover:shadow-md transition-shadow">
                        <div class="flex justify-between items-start">
                          <div>
                            <a-space align="center">
                              <span class="font-bold text-base">{{ tpl.title }}</span>
                              <a-tag color="blue">{{ tpl.type }}</a-tag>
                            </a-space>
                            <div class="mt-2 text-gray-500 text-sm">{{ tpl.description }}</div>
                          </div>
                          <a-button type="primary" @click="selectedTemplate = tpl; currentView = 'create'; selectedEvents = []">发起流程</a-button>
                        </div>
                      </a-card>
                    </a-col>
                  </a-row>
                </div>
              </div>

              <!-- Create Order -->
              <div v-else-if="currentView === 'create'" class="h-full flex flex-col space-y-4">
                <div class="bg-white p-4 rounded-sm shadow-sm flex items-center gap-4">
                  <a-button type="text" @click="currentView = 'template'"><template #icon><ArrowLeftOutlined /></template>返回</a-button>
                  <span class="font-medium text-base">填写工单信息</span>
                </div>
                <div class="bg-white p-8 rounded-sm shadow-sm flex-1 overflow-y-auto">
                  <div class="max-w-4xl mx-auto w-full">
                    <a-form layout="vertical">
                      <a-form-item label="工单名称" required>
                        <a-input v-model:value="orderForm.name" placeholder="请输入工单名称" />
                      </a-form-item>
                      <a-form-item label="紧急程度">
                        <a-radio-group v-model:value="orderForm.priority">
                          <a-radio-button value="high">高</a-radio-button>
                          <a-radio-button value="medium">中</a-radio-button>
                          <a-radio-button value="low">低</a-radio-button>
                        </a-radio-group>
                      </a-form-item>
                      
                      <a-form-item label="备注">
                        <a-textarea placeholder="请输入备注信息" :rows="4" />
                      </a-form-item>

                      <div v-if="selectedTemplate?.title === '事件处置'" class="mb-6">
                        <div class="flex items-center justify-between mb-2">
                          <span class="font-medium text-gray-700">事件信息</span>
                          <a-button type="primary" ghost size="small" @click="tempSelectedEvents = [...selectedEvents]; isEventModalVisible = true">
                            <template #icon><PlusOutlined /></template>添加事件
                          </a-button>
                        </div>
                        <div v-if="selectedEvents.length > 0" class="border rounded-sm overflow-hidden">
                          <a-table :dataSource="selectedEvents" :pagination="false" size="small" rowKey="id">
                            <a-table-column title="事件名称" dataIndex="name" />
                            <a-table-column title="影响资产" dataIndex="asset" />
                            <a-table-column title="来源平台" dataIndex="platform" />
                            <a-table-column title="等级" dataIndex="level">
                              <template #default="{ text }">
                                <a-tag :color="text === '严重' ? 'red' : text === '高危' ? 'volcano' : text === '中危' ? 'orange' : 'blue'">{{ text }}</a-tag>
                              </template>
                            </a-table-column>
                            <a-table-column title="操作" key="action">
                              <template #default="{ record }">
                                <a-button type="link" danger size="small" @click="selectedEvents = selectedEvents.filter(e => e.id !== record.id)">移除</a-button>
                              </template>
                            </a-table-column>
                          </a-table>
                        </div>
                        <div v-else class="border border-dashed rounded-sm p-8 text-center text-gray-400 bg-gray-50">
                          暂未关联事件，请点击上方按钮添加
                        </div>
                      </div>

                      <a-divider orientation="left">指派下一步责任人</a-divider>
                      <a-form-item label="责任人" required>
                        <div 
                          class="ant-input cursor-pointer hover:border-[#0052D9] transition-colors flex flex-wrap gap-2 items-center min-h-[32px] px-2 py-1"
                          @click="tempResponsible = [...orderForm.responsibleUnits]; isResponsibleModalVisible = true"
                        >
                          <template v-if="orderForm.responsibleUnits.length > 0">
                            <a-tag v-for="u in orderForm.responsibleUnits" :key="u" closable @close.stop="orderForm.responsibleUnits = orderForm.responsibleUnits.filter(i => i !== u)">
                              {{ u }}
                            </a-tag>
                          </template>
                          <span v-else class="text-gray-400">点击选择责任人</span>
                        </div>
                      </a-form-item>
                      
                      <div class="mt-8 pt-8 border-t flex justify-end gap-4">
                        <a-button @click="currentView = 'template'">取消</a-button>
                        <a-button type="primary" @click="handleCreateOrder">确认发起</a-button>
                      </div>
                    </a-form>
                  </div>
                </div>
              </div>

              <!-- Detail View -->
              <div v-else-if="currentView === 'detail' && selectedOrder" class="h-full flex flex-col">
                <div class="flex-1 overflow-hidden relative">
                  <div class="absolute inset-0 overflow-y-auto p-4 space-y-4">
                    <div class="bg-white p-4 rounded-sm shadow-sm flex items-center justify-between">
                      <div class="flex items-center gap-4">
                        <span class="font-bold text-lg">{{ selectedOrder.name }}</span>
                        <a-tag color="blue">{{ selectedOrder.type }}</a-tag>
                        <span :class="selectedOrder.flowStatus === 'completed' ? 'text-green-500' : 'text-black'" class="text-sm">
                          {{ (selectedOrder.flowStatus === 'processing' || selectedOrder.flowStatus === 'timeout') ? '流转中' : '已结束' }}
                        </span>
                      </div>
                    </div>

                    <div class="bg-white p-6 rounded-sm shadow-sm">
                      <a-descriptions title="基础信息" :column="3" bordered size="small">
                        <a-descriptions-item label="工单编号">{{ selectedOrder.id }}</a-descriptions-item>
                        <a-descriptions-item label="紧急程度">
                          <a-tag :color="selectedOrder.priority === 'high' ? 'red' : selectedOrder.priority === 'medium' ? 'orange' : 'blue'">
                            {{ selectedOrder.priority === 'high' ? '高' : selectedOrder.priority === 'medium' ? '中' : '低' }}
                          </a-tag>
                        </a-descriptions-item>
                        <a-descriptions-item label="当前状态">
                          <span :class="(selectedOrder.flowStatus === 'processing' || selectedOrder.flowStatus === 'timeout') ? 'text-black' : 'text-green-500'">
                            {{ (selectedOrder.flowStatus === 'processing' || selectedOrder.flowStatus === 'timeout') ? '流转中' : '已结束' }}
                          </span>
                        </a-descriptions-item>
                        <a-descriptions-item label="发起人">{{ selectedOrder.creator }}</a-descriptions-item>
                        <a-descriptions-item label="创建时间">{{ selectedOrder.createTime }}</a-descriptions-item>
                        <a-descriptions-item label="更新时间">{{ selectedOrder.updateTime }}</a-descriptions-item>
                      </a-descriptions>
                    </div>

                    <div v-if="selectedOrder.template === '事件处置'" class="bg-white p-6 rounded-sm shadow-sm">
                      <div class="mb-4 font-bold text-base">关联事件</div>
                      <a-table :dataSource="[
                        { id: '1', name: '主机存在RCE利用、反弹Shell', asset: '172.23.162.163', platform: '级联下级平台', level: '低危', time: '2024-04-14 10:00:00' }
                      ]" :pagination="false" size="small">
                        <a-table-column title="事件名称" dataIndex="name" />
                        <a-table-column title="影响资产" dataIndex="asset" />
                        <a-table-column title="来源平台" dataIndex="platform" />
                        <a-table-column title="等级" dataIndex="level">
                          <template #default="{ text }">
                            <a-tag color="blue">{{ text }}</a-tag>
                          </template>
                        </a-table-column>
                        <a-table-column title="发生时间" dataIndex="time" />
                      </a-table>
                    </div>

                    <div class="bg-white p-6 rounded-sm shadow-sm">
                      <div class="mb-6 flex justify-between items-center">
                        <div class="flex items-center gap-2">
                          <span class="font-bold text-base">流程进度</span>
                          <div v-if="selectedOrder.hasFailure" class="flex items-center gap-2 bg-orange-50 px-3 py-1 rounded border border-orange-100">
                            <InfoCircleOutlined class="text-orange-400" />
                            <span class="text-xs text-orange-600">当前共有 2 个工单发送失败， 4 个工单数据获取失败。</span>
                            <a-button type="link" size="small" class="p-0 h-auto text-xs" :loading="isRetrying" @click="handleRetry">重新尝试</a-button>
                          </div>
                        </div>
                      </div>
                      <div class="px-4">
                        <!-- Stage 1 -->
                        <div class="relative pl-8 border-l-2 border-blue-500 ml-4 pb-8">
                          <div class="absolute -left-[11px] top-0 w-5 h-5 bg-blue-500 rounded-full flex items-center justify-center text-white text-xs">1</div>
                          <div class="bg-gray-50 p-4 rounded-sm border border-[#f0f0f0]">
                            <div class="flex justify-between items-center mb-4">
                              <span class="font-bold text-base">阶段：发起工单</span>
                              <a-tag color="green">已完成</a-tag>
                            </div>
                            <a-descriptions :column="2" size="small">
                              <a-descriptions-item label="操作人">{{ selectedOrder.creator }}</a-descriptions-item>
                              <a-descriptions-item label="操作时间">{{ selectedOrder.createTime }}</a-descriptions-item>
                              <a-descriptions-item label="发起内容" :span="2">全网漏洞专项排查，请各单位配合。</a-descriptions-item>
                            </a-descriptions>
                          </div>
                        </div>

                        <!-- Stage 2 -->
                        <div v-if="selectedOrder.subPlatforms.some(s => s.status !== 'approval' && s.status !== 'completed')" class="relative pl-8 border-l-2 border-blue-500 ml-4 pb-8">
                          <div class="absolute -left-[11px] top-0 w-5 h-5 bg-blue-500 rounded-full flex items-center justify-center text-white text-xs">2</div>
                          <div class="bg-white p-4 rounded-sm shadow-sm border border-[#f0f0f0]">
                            <div class="flex justify-between items-center mb-4 cursor-pointer" @click="toggleStage('disposal')">
                              <div class="flex items-center gap-2">
                                <span class="font-bold text-base">阶段：处置</span>
                                <LinkOutlined class="text-gray-400" />
                                <DownOutlined :rotate="expandedStages.includes('disposal') ? 0 : -90" class="text-xs text-gray-400 transition-transform ml-2" />
                              </div>
                              <a-space>
                                <a-radio-group v-model:value="disposalFilter" size="small" @click.stop>
                                  <a-radio-button value="all">全部 ({{ getDisposalCounts.all }})</a-radio-button>
                                  <a-radio-button value="processing">流转中 ({{ getDisposalCounts.processing }})</a-radio-button>
                                  <a-radio-button value="timeout">已超时 ({{ getDisposalCounts.timeout }})</a-radio-button>
                                  <a-radio-button v-if="!selectedOrder.hasApprovalStage" value="completed">已结束 ({{ getDisposalCounts.completed }})</a-radio-button>
                                </a-radio-group>
                              </a-space>
                            </div>
                            
                            <div v-show="expandedStages.includes('disposal')" v-if="selectedOrder.isCascading" class="space-y-4">
                              <a-table
                                :pagination="false"
                                size="small"
                                :dataSource="[...filteredSubPlatforms].sort((a, b) => (a.isLocal ? -1 : b.isLocal ? 1 : 0))"
                                rowKey="name"
                              >
                                <a-table-column title="接收单位" dataIndex="name">
                                  <template #default="{ text, record }">
                                    <div class="flex items-center gap-2 group">
                                      <span>{{ text }}</span>
                                      <a-tag v-if="record.isLocal" color="blue" size="small">本平台</a-tag>
                                      <a-tooltip title="模拟下级平台完成处置">
                                        <ReloadOutlined 
                                          class="text-blue-400 cursor-pointer hidden group-hover:inline-block" 
                                          @click="simulateSubPlatformComplete(record)"
                                        />
                                      </a-tooltip>
                                    </div>
                                  </template>
                                </a-table-column>
                                <a-table-column title="当前状态" dataIndex="status">
                                  <template #default="{ text, record }">
                                    <span :class="record.isProcessed ? 'text-orange-500' : (text === 'completed' ? 'text-green-500' : 'text-black')">
                                      {{ record.isProcessed ? '待审核' : ((text === 'processing' || text === 'timeout') ? '流转中' : '已完成') }}
                                    </span>
                                  </template>
                                </a-table-column>
                                <a-table-column title="耗时" dataIndex="timeConsumed" />
                                <a-table-column title="操作" key="action">
                                  <template #default="{ record, index }">
                                    <a-space>
                                      <a-button v-if="record.isLocal && !record.isProcessed" type="link" size="small" class="p-0" @click="currentSubPlatform = record; isProcessDrawerVisible = true">处理工单</a-button>
                                      <a-button v-if="record.isLocal && record.isProcessed" type="link" size="small" class="p-0" @click="currentSubPlatform = record; isProcessDrawerVisible = true">修改</a-button>
                                      <a-button type="link" size="small" class="p-0" @click="showFlowDrawer(record, index)">详情</a-button>
                                      <a-button v-if="record.status === 'timeout'" type="link" size="small" danger class="p-0" @click="handleUrge(record)">催办</a-button>
                                    </a-space>
                                  </template>
                                </a-table-column>
                              </a-table>
                            </div>
                            <div v-show="expandedStages.includes('disposal')" v-else class="bg-gray-50 p-4 rounded">
                              <span class="text-gray-400">普通处置节点，等待责任人完成任务...</span>
                            </div>
                          </div>
                        </div>

                        <!-- Stage 3 (Conditional) -->
                        <div v-if="selectedOrder.hasApprovalStage" class="relative pl-8 border-l-2 border-gray-200 ml-4 pb-4">
                          <div class="absolute -left-[11px] top-0 w-5 h-5 bg-gray-300 rounded-full flex items-center justify-center text-white text-xs">3</div>
                          <div class="bg-white p-4 rounded-sm shadow-sm border border-[#f0f0f0]">
                            <div class="flex justify-between items-center mb-4 cursor-pointer" @click="toggleStage('approval')">
                              <div class="flex items-center gap-2">
                                <span class="font-bold text-base">阶段：审核</span>
                                <a-tag v-if="getApprovalCounts.approval > 0" color="orange">待审核</a-tag>
                                <DownOutlined :rotate="expandedStages.includes('approval') ? 0 : -90" class="text-xs text-gray-400 transition-transform ml-2" />
                              </div>
                              <a-space>
                                <a-radio-group v-model:value="approvalFilter" size="small" @click.stop>
                                  <a-radio-button value="all">全部 ({{ getApprovalCounts.all }})</a-radio-button>
                                  <a-radio-button value="approval">待审核 ({{ getApprovalCounts.approval }})</a-radio-button>
                                  <a-radio-button value="completed">已结束 ({{ getApprovalCounts.completed }})</a-radio-button>
                                </a-radio-group>
                              </a-space>
                            </div>
                            
                            <div v-show="expandedStages.includes('approval')">
                              <a-table
                                v-if="selectedOrder.subPlatforms.some(s => s.status === 'approval' || s.status === 'completed')"
                                :pagination="false"
                                size="small"
                                :dataSource="[...selectedOrder.subPlatforms.filter(s => s.status === 'approval' || s.status === 'completed')].sort((a, b) => (a.isLocal ? -1 : b.isLocal ? 1 : 0))"
                                rowKey="name"
                              >
                                <a-table-column title="接收单位" dataIndex="name">
                                  <template #default="{ text, record }">
                                    <div class="flex items-center gap-2">
                                      <span>{{ text }}</span>
                                      <a-tag v-if="record.isLocal" color="blue" size="small">本平台</a-tag>
                                    </div>
                                  </template>
                                </a-table-column>
                                <a-table-column title="当前状态" dataIndex="status">
                                  <template #default="{ text }">
                                    <span :class="text === 'completed' ? 'text-green-500' : 'text-orange-500'">
                                      {{ text === 'approval' ? '待审核' : '已结束' }}
                                    </span>
                                  </template>
                                </a-table-column>
                                <a-table-column title="耗时" dataIndex="timeConsumed" />
                                <a-table-column title="操作" key="action">
                                  <template #default="{ record, index }">
                                    <a-space>
                                      <a-button v-if="record.status === 'approval'" type="link" size="small" class="p-0" @click="showFlowDrawer(record, index)">审核</a-button>
                                      <a-button v-else type="link" size="small" class="p-0" @click="showFlowDrawer(record, index)">详情</a-button>
                                    </a-space>
                                  </template>
                                </a-table-column>
                              </a-table>
                              <a-empty v-else description="暂无待审核项" :image="Empty.PRESENTED_IMAGE_SIMPLE" />
                            </div>
                          </div>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
                <!-- Bottom Action Bar -->
                <div class="bg-white p-4 border-t flex justify-end gap-3">
                  <a-button @click="currentView = 'list'">关闭工单</a-button>
                  <a-button @click="message.success('导出成功')">导出</a-button>
                </div>
              </div>

              <!-- Dept Management -->
              <div v-else-if="currentView === 'dept'" class="h-full bg-white rounded-sm shadow-sm flex overflow-hidden">
                <div class="w-96 border-r border-[#f0f0f0] flex flex-col">
                  <div class="p-4 border-b border-[#f0f0f0] flex justify-between items-center bg-gray-50/50">
                    <span class="font-medium">部门</span>
                    <a-button type="link" size="small" @click="isAddDeptModalVisible = true">新增</a-button>
                  </div>
                  <div class="p-3 border-b border-[#f0f0f0]">
                    <a-input placeholder="搜索部门" size="small">
                      <template #prefix><SearchOutlined class="text-gray-400" /></template>
                    </a-input>
                  </div>
                  <div class="flex-1 overflow-y-auto p-2">
                    <a-tree
                      defaultExpandAll
                      :treeData="cleanDepartments"
                      @select="handleDeptSelect"
                      class="ant-tree-custom"
                    >
                      <template #title="node">
                        <div class="flex items-center justify-between group w-full pr-2">
                          <div class="flex items-center gap-2 overflow-hidden">
                            <ClusterOutlined v-if="node.isCascading" class="text-blue-500 shrink-0" />
                            <span class="text-sm truncate">{{ node.title }}</span>
                          </div>
                        </div>
                      </template>
                    </a-tree>
                  </div>
                </div>
                <div class="flex-1 flex flex-col overflow-hidden">
                  <div class="p-4 border-b border-[#f0f0f0] flex justify-between items-center">
                    <a-space v-if="!isCascadingSelected">
                      <a-button type="primary" @click="isAddMemberModalVisible = true">新增</a-button>
                      <a-button>移动</a-button>
                      <a-button disabled>删除</a-button>
                      <a-button>导入</a-button>
                    </a-space>
                    <div v-else class="text-gray-400 text-sm">
                      <InfoCircleOutlined class="mr-1" /> 该部门为级联同步部门，不支持手动管理人员
                    </div>
                    <div class="flex items-center gap-2">
                      <a-input-search placeholder="请搜索姓名" style="width: 200px" />
                      <a-button><template #icon><ReloadOutlined /></template></a-button>
                    </div>
                  </div>
                  <div class="flex-1 overflow-y-auto p-4">
                    <a-table 
                      v-if="filteredMembers.length > 0"
                      rowKey="key"
                      :dataSource="filteredMembers"
                      :pagination="{ showSizeChanger: true, showQuickJumper: true, total: filteredMembers.length, size: 'small' }"
                      size="small"
                      :row-selection="{}"
                    >
                      <a-table-column title="序号" key="index" width="60">
                        <template #default="{ index }">{{ index + 1 }}</template>
                      </a-table-column>
                      <a-table-column title="姓名" dataIndex="name" />
                      <a-table-column title="所属部门" dataIndex="deptPath" />
                      <a-table-column title="是否主管" key="isManager">
                        <template #default="{ record }">
                          {{ record.key === '1' ? '是' : '否' }}
                        </template>
                      </a-table-column>
                      <a-table-column title="职位" dataIndex="role" />
                      <a-table-column title="操作" key="action" width="120">
                        <template #default>
                          <a-space>
                            <a-button type="link" size="small" class="p-0" :disabled="isCascadingSelected">编辑</a-button>
                            <a-button type="link" size="small" danger class="p-0" :disabled="isCascadingSelected">删除</a-button>
                          </a-space>
                        </template>
                      </a-table-column>
                    </a-table>
                    <div v-else class="h-full flex items-center justify-center">
                      <a-empty>
                        <template #description>
                          <div v-if="isCascadingSelected" class="max-w-md text-gray-400">
                            下级平台暂未同步工单人员，请在下级平台工单模块的 “上级平台” 部门下新增成员，完成后自动同步。
                          </div>
                          <div v-else class="text-gray-400">暂无数据</div>
                        </template>
                      </a-empty>
                    </div>
                  </div>
                </div>
              </div>

            </div>
          </a-layout-content>
        </a-layout>
      </a-layout>
    </div>

    <!-- Modals -->
    <a-modal
      v-model:open="isAddDeptModalVisible"
      title="新增部门"
      @ok="handleAddDept"
    >
      <a-form layout="vertical">
        <a-form-item label="所属部门">
          <a-tree-select
            v-model:value="deptForm.parent"
            :treeData="cleanDepartments"
            placeholder="请选择"
            treeDefaultExpandAll
          />
        </a-form-item>
        <a-form-item label="部门名称">
          <a-input v-model:value="deptForm.name" placeholder="请输入" />
        </a-form-item>
        <a-form-item label="关联下级平台">
          <a-select v-model:value="deptForm.cascadingPlatform" placeholder="请选择下级平台" allowClear>
            <a-select-option v-for="p in cascadingPlatforms" :key="p" :value="p">{{ p }}</a-select-option>
          </a-select>
        </a-form-item>
      </a-form>
    </a-modal>

    <a-modal
      v-model:open="isEditDeptModalVisible"
      title="编辑部门"
      @ok="saveEditDept"
    >
      <a-form layout="vertical">
        <a-form-item label="部门名称">
          <a-input v-model:value="deptForm.name" placeholder="请输入" />
        </a-form-item>
      </a-form>
    </a-modal>

    <a-modal
      v-model:open="isAddMemberModalVisible"
      title="新增成员"
      @ok="isAddMemberModalVisible = false; message.success('成员添加成功')"
    >
      <a-form layout="vertical">
        <a-form-item label="姓名" required>
          <a-input placeholder="请输入姓名" />
        </a-form-item>
        <a-form-item label="所属部门" required>
          <a-tree-select
            :treeData="cleanDepartments"
            placeholder="请选择部门"
            treeDefaultExpandAll
          />
        </a-form-item>
      </a-form>
    </a-modal>

    <a-modal
      v-model:open="isResponsibleModalVisible"
      title="选择责任人"
      :width="800"
      @ok="orderForm.responsibleUnits = [...tempResponsible]; isResponsibleModalVisible = false"
      :bodyStyle="{ padding: '0' }"
    >
      <div class="flex h-[500px]">
        <div class="w-1/2 border-r flex flex-col overflow-hidden">
          <div class="p-4 border-b">
            <a-tabs size="small" class="mb-4">
              <a-tab-pane key="dept" tab="部门" />
              <a-tab-pane key="group" tab="工作组" />
              <a-tab-pane key="role" tab="角色" />
              <a-tab-pane key="account" tab="账号" />
            </a-tabs>
            <a-input placeholder="搜索关键字">
              <template #prefix><SearchOutlined class="text-gray-400" /></template>
            </a-input>
          </div>
          <div class="flex-1 overflow-y-auto p-4">
            <a-tree
              checkable
              defaultExpandAll
              :treeData="treeDataWithDisabled"
              :checkedKeys="tempResponsible.map(t => findKeyByTitle(departments, t) || '')"
              @check="onCheck"
              class="ant-tree-custom"
            >
              <template #title="node">
                <div class="flex items-center gap-2 group">
                  <ClusterOutlined v-if="node.isCascading" class="text-blue-500" />
                  <a-tooltip :title="node.tooltip">
                    <span :class="{ 'text-gray-300': node.disableCheckbox }">{{ node.title }}</span>
                  </a-tooltip>
                  <a-tooltip v-if="node.disableCheckbox" :title="node.tooltip">
                    <InfoCircleOutlined class="text-gray-400" />
                  </a-tooltip>
                </div>
              </template>
              <template #switcherIcon>
                <NodeIndexOutlined />
              </template>
            </a-tree>
          </div>
        </div>
        <div class="w-1/2 flex flex-col overflow-hidden bg-gray-50/30">
          <div class="p-4 border-b flex justify-between items-center">
            <span class="font-medium">已选择 ({{ tempResponsible.length }})</span>
            <a-button type="link" size="small" @click="tempResponsible = []" danger>清空</a-button>
          </div>
          <div class="flex-1 overflow-y-auto p-4 flex flex-wrap gap-2 content-start">
            <a-tag v-for="u in tempResponsible" :key="u" closable @close="tempResponsible = tempResponsible.filter(i => i !== u)" class="m-0">
              {{ u }}
            </a-tag>
          </div>
        </div>
      </div>
    </a-modal>

    <!-- Select Event Modal -->
    <a-modal
      v-model:open="isEventModalVisible"
      title="选择事件"
      :width="1100"
      @ok="selectedEvents = [...tempSelectedEvents]; isEventModalVisible = false"
      :bodyStyle="{ padding: '0' }"
    >
      <div class="flex h-[600px]">
        <div class="w-2/3 border-r flex flex-col overflow-hidden">
          <div class="p-4 border-b space-y-4">
            <div class="flex gap-2">
              <a-range-picker style="width: 240px" />
              <a-select placeholder="全部事件等级" style="width: 140px">
                <a-select-option value="high">严重/高危</a-select-option>
              </a-select>
              <a-select v-model:value="eventPlatformFilter" placeholder="全部来源平台" style="width: 140px">
                <a-select-option value="all">全部来源平台</a-select-option>
                <a-select-option value="本平台">本平台</a-select-option>
                <a-select-option value="级联下级平台">级联下级平台</a-select-option>
              </a-select>
              <a-input-search placeholder="搜索事件名称" style="width: 200px" />
            </div>
          </div>
          <div class="flex-1 overflow-y-auto">
            <a-table 
              :dataSource="filteredEventList" 
              rowKey="id" 
              size="small" 
              :pagination="{ total: 26939, size: 'small' }"
              :rowSelection="{
                selectedRowKeys: tempSelectedEvents.map(e => e.id),
                onChange: (keys, rows) => { tempSelectedEvents = rows }
              }"
            >
              <a-table-column title="处置状态" dataIndex="status" />
              <a-table-column title="结束时间" dataIndex="time" />
              <a-table-column title="事件名称" dataIndex="name" />
              <a-table-column title="影响资产" dataIndex="asset" />
              <a-table-column title="来源平台" dataIndex="platform" />
              <a-table-column title="事件等级" dataIndex="level">
                <template #default="{ text }">
                  <a-tag :color="text === '严重' ? 'red' : text === '高危' ? 'volcano' : text === '中危' ? 'orange' : 'blue'">{{ text }}</a-tag>
                </template>
              </a-table-column>
            </a-table>
          </div>
        </div>
        <div class="w-1/3 flex flex-col overflow-hidden bg-gray-50/30">
          <div class="p-4 border-b flex justify-between items-center">
            <span class="font-medium">已选 ({{ tempSelectedEvents.length }})</span>
            <a-button type="link" size="small" @click="tempSelectedEvents = []" danger>清空</a-button>
          </div>
          <div class="flex-1 overflow-y-auto p-4">
            <div v-if="tempSelectedEvents.length > 0" class="space-y-2">
              <div v-for="e in tempSelectedEvents" :key="e.id" class="bg-white p-2 border rounded-sm flex justify-between items-center group">
                <div class="overflow-hidden">
                  <div class="text-sm font-medium truncate">{{ e.name }}</div>
                  <div class="text-xs text-gray-400">{{ e.time }}</div>
                </div>
                <a-button type="link" size="small" danger @click="tempSelectedEvents = tempSelectedEvents.filter(i => i.id !== e.id)">移除</a-button>
              </div>
            </div>
            <a-empty v-else description="暂无数据" />
          </div>
        </div>
      </div>
    </a-modal>

    <!-- Flow Detail Drawer -->
    <a-drawer
      v-model:open="isProcessDrawerVisible"
      title="处理工单"
      width="800"
      placement="right"
    >
      <div class="flex flex-col h-full">
        <div class="flex-1 overflow-y-auto">
          <a-form layout="vertical">
            <a-form-item label="处置结果" required>
              <a-radio-group v-model:value="processForm.result">
                <a-radio value="close">关闭</a-radio>
                <a-radio value="transfer">转交</a-radio>
              </a-radio-group>
            </a-form-item>
            <a-form-item label="描述">
              <a-textarea v-model:value="processForm.desc" placeholder="请输入描述信息" :rows="6" />
            </a-form-item>
            <a-form-item label="相关文件">
              <a-upload>
                <a-button><template #icon><PlusOutlined /></template>上传文件</a-button>
              </a-upload>
              <div class="text-xs text-gray-400 mt-2">支持txt、pdf、doc、docx、xls、xlsx、xml、ppt、pptx、wps、html、jpg、jpeg、png、gif、bmp、rar、zip、7z</div>
            </a-form-item>
            
            <a-divider orientation="left">指派下一步责任人</a-divider>
            <a-form-item label="抄送">
              <a-select v-model:value="processForm.cc" mode="multiple" placeholder="请选择人员" style="width: 100%">
                <a-select-option v-for="m in allMembers" :key="m.key" :value="m.name">{{ m.name }}</a-select-option>
              </a-select>
            </a-form-item>
          </a-form>
        </div>
        <div class="pt-4 border-t flex justify-end gap-3">
          <a-button @click="isProcessDrawerVisible = false">取消</a-button>
          <a-button type="primary" @click="handleProcessSubmit">确定</a-button>
        </div>
      </div>
    </a-drawer>

    <!-- Flow Detail Drawer -->
    <a-drawer
      v-model:open="isFlowDrawerVisible"
      :title="`流转详情 - ${currentSubPlatform?.name}`"
      width="600"
      placement="right"
    >
      <div class="flex flex-col h-full">
        <div v-if="currentSubPlatform?.status === 'completed'" class="mb-4">
          <a-alert message="工单已结束" type="success" show-icon />
        </div>
        
        <div class="flex-1 overflow-y-auto pr-4 py-4">
          <a-timeline>
            <template v-for="(h, i) in currentSubPlatform?.flowHistory" :key="i">
              <a-timeline-item :color="h.status === '已驳回' ? 'red' : h.status === '审批通过' ? 'green' : 'blue'">
                <template #dot v-if="h.node === '催办'">
                  <InfoCircleOutlined class="text-red-500" />
                </template>
                <div class="flex flex-col gap-2">
                  <div class="flex justify-between items-center">
                    <span class="font-bold" :class="{'text-red-500': h.status === '已驳回'}">{{ h.node }} - {{ h.status }}</span>
                    <span class="text-gray-400 text-xs">{{ h.time }}</span>
                  </div>
                  <div class="bg-white p-3 rounded shadow-sm border border-[#f0f0f0]" :class="{'border-red-100 bg-red-50/30': h.node === '催办' || h.status === '已驳回'}">
                    <div class="text-xs text-gray-500 mb-1">{{ h.node === '催办' ? '催办人' : '责任人' }}</div>
                    <div class="text-sm">{{ h.responsible }}</div>
                    <div v-if="h.content" class="mt-2 text-sm text-gray-600 italic border-t pt-2 border-gray-100">
                      {{ h.content }}
                    </div>
                    <div v-if="h.attachment" class="mt-2 p-2 bg-gray-50 rounded border border-gray-100 flex items-center justify-between group">
                      <div class="flex items-center gap-2 overflow-hidden">
                        <FileTextOutlined class="text-blue-500 shrink-0" />
                        <span class="text-xs truncate">{{ h.attachment.name }}</span>
                      </div>
                      <a-button type="link" size="small" class="p-0 h-auto text-xs">下载</a-button>
                    </div>
                  </div>
                </div>
              </a-timeline-item>
            </template>

            <!-- Current Status Node -->
            <a-timeline-item v-if="currentSubPlatform?.status !== 'completed'" :color="currentSubPlatform?.status === 'timeout' ? 'red' : 'blue'">
              <template #dot v-if="currentSubPlatform?.status === 'timeout'">
                <ClockCircleOutlined class="text-red-500" />
              </template>
              <div class="flex flex-col gap-2">
                <div class="flex justify-between items-center">
                  <div class="flex items-center gap-2">
                    <span class="font-bold">{{ currentSubPlatform?.status === 'approval' ? '等待审核中' : '等待处理中' }}</span>
                    <a-tag v-if="currentSubPlatform?.status === 'timeout'" class="bg-gray-100 text-black border-gray-200" size="small">流转中</a-tag>
                  </div>
                </div>
                
                <!-- Approval Form if pending approval -->
                <div v-if="currentSubPlatform?.status === 'approval'" class="bg-white p-4 rounded shadow-sm border border-[#f0f0f0]">
                  <a-form layout="vertical">
                    <a-form-item label="审核结果" required>
                      <a-radio-group v-model:value="approvalResult">
                        <a-radio value="pass">通过</a-radio>
                        <a-radio value="reject">驳回</a-radio>
                      </a-radio-group>
                    </a-form-item>
                    <a-form-item label="审核意见">
                      <a-textarea placeholder="请输入审核意见" :rows="2" />
                    </a-form-item>
                    <a-button type="primary" block @click="submitApproval">提交审核</a-button>
                  </a-form>
                </div>
                <div v-else class="bg-white p-3 rounded shadow-sm border border-[#f0f0f0] border-dashed">
                  <div class="text-xs text-gray-400">当前节点正在处理中...</div>
                </div>
              </div>
            </a-timeline-item>
          </a-timeline>
        </div>

        <div class="pt-4 border-t flex justify-between items-center">
          <a-space>
            <a-button @click="handlePrevNext('prev')" :disabled="currentOrderIndex === 0">上一条</a-button>
            <a-button @click="handlePrevNext('next')" :disabled="!selectedOrder || currentOrderIndex === selectedOrder.subPlatforms.length - 1">下一条</a-button>
          </a-space>
          <a-space>
            <a-button @click="isFlowDrawerVisible = false">关闭</a-button>
          </a-space>
        </div>
      </div>
    </a-drawer>

  </a-config-provider>
</template>

<style>
@import "tailwindcss";

body {
  margin: 0;
  padding: 0;
  background-color: #f0f2f5;
}

/* Ant Design Vue overrides */
.ant-layout {
  background: transparent !important;
}

.ant-menu-dark.ant-menu-inline .ant-menu-sub.ant-menu-inline {
  background: rgba(0, 0, 0, 0.2) !important;
}

.ant-table-wrapper .ant-table-thead > tr > th {
  background: #fafafa !important;
  font-weight: 500 !important;
  color: rgba(0, 0, 0, 0.85) !important;
}

.ant-tree-custom .ant-tree-node-content-wrapper {
  padding: 4px 8px !important;
}

.ant-tree-custom .ant-tree-treenode-selected .ant-tree-node-content-wrapper {
  background-color: #e6f7ff !important;
}

.ant-descriptions-item-label {
  color: #8c8c8c !important;
  background-color: #fafafa !important;
}

.ant-modal-header {
  border-bottom: 1px solid #f0f0f0 !important;
  padding: 16px 24px !important;
}

.ant-modal-footer {
  border-top: 1px solid #f0f0f0 !important;
  padding: 10px 16px !important;
}
</style>
