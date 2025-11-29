import React, { useState, useMemo } from 'react';
import { 
  LayoutDashboard, 
  Users, 
  Settings, 
  Calculator, 
  TrendingUp, 
  DollarSign, 
  ChevronRight, 
  ChevronDown, 
  Plus, 
  Search,
  ArrowUp,
  Award,
  Wallet,
  FileText
} from 'lucide-react';
import { 
  BarChart, 
  Bar, 
  XAxis, 
  YAxis, 
  CartesianGrid, 
  Tooltip, 
  ResponsiveContainer,
  PieChart,
  Pie,
  Cell
} from 'recharts';

// --- 模拟数据 ---

// 1. 等级配置 (级差核心)
const INITIAL_LEVELS = [
  { id: 'L1', name: '实习代理', rate: 10, color: '#94a3b8' }, // 10%
  { id: 'L2', name: '正式经理', rate: 20, color: '#3b82f6' }, // 20%
  { id: 'L3', name: '高级总监', rate: 30, color: '#8b5cf6' }, // 30%
  { id: 'L4', name: '区域合伙人', rate: 40, color: '#eab308' }, // 40%
  { id: 'L5', name: '全球董事', rate: 45, color: '#ef4444' }, // 45% (拨比上限)
];

// 2. 代理团队数据 (树状结构扁平化存储)
const INITIAL_AGENTS = [
  { id: 1, name: '张三 (总董)', level: 'L5', parentId: null, sales: 1200000, teamSize: 45 },
  { id: 2, name: '李四 (华东)', level: 'L4', parentId: 1, sales: 500000, teamSize: 20 },
  { id: 3, name: '王五 (华南)', level: 'L4', parentId: 1, sales: 400000, teamSize: 15 },
  { id: 4, name: '赵六 (上海)', level: 'L3', parentId: 2, sales: 200000, teamSize: 8 },
  { id: 5, name: '钱七 (杭州)', level: 'L3', parentId: 2, sales: 150000, teamSize: 5 },
  { id: 6, name: '孙八 (深圳)', level: 'L3', parentId: 3, sales: 300000, teamSize: 10 },
  { id: 7, name: '周九 (业务员)', level: 'L1', parentId: 4, sales: 50000, teamSize: 0 },
  { id: 8, name: '吴十 (业务员)', level: 'L2', parentId: 4, sales: 80000, teamSize: 2 },
];

// 3. 最近订单/业绩记录
const RECENT_ORDERS = [
  { id: 'ORD-001', agentName: '周九 (业务员)', amount: 10000, date: '2023-10-24' },
  { id: 'ORD-002', agentName: '吴十 (业务员)', amount: 5000, date: '2023-10-24' },
  { id: 'ORD-003', agentName: '孙八 (深圳)', amount: 50000, date: '2023-10-23' },
];

// --- 工具函数 ---

const formatCurrency = (val) => new Intl.NumberFormat('zh-CN', { style: 'currency', currency: 'CNY' }).format(val);

// 级差计算核心逻辑
// 递归向上查找上级，并计算级差
const calculateCommissions = (orderAmount, agentId, allAgents, allLevels) => {
  const breakdown = [];
  let currentAgent = allAgents.find(a => a.id === agentId);
  let lastRate = 0; // 下级的比例 (级差是被减数)

  while (currentAgent) {
    const levelConfig = allLevels.find(l => l.id === currentAgent.level);
    const currentRate = levelConfig ? levelConfig.rate : 0;
    
    // 级差 = 当前等级比例 - 下级已拿走的比例
    // 如果当前等级比例 <= 下级比例，则出现“平级”或“倒挂”，通常无级差或仅有平级奖(此处简化为0)
    let diffRate = currentRate - lastRate;
    if (diffRate < 0) diffRate = 0; 

    const bonus = orderAmount * (diffRate / 100);

    if (bonus > 0) {
      breakdown.push({
        agent: currentAgent.name,
        level: levelConfig.name,
        roleRate: `${currentRate}%`,
        diffRate: `${diffRate.toFixed(1)}%`,
        calculation: `${currentRate}% - ${lastRate}%`,
        amount: bonus
      });
    }

    // 更新 lastRate 为当前等级，供再上一级计算差值
    // 注意：级差制中，通常是“紧缩”计算。如果上级比例比下级低，lastRate 应该保持 max(lastRate, currentRate) 吗？
    // 标准级差逻辑：每层只拿自己这层多出来的部分。如果上级比下级低，上级拿不到钱，且lastRate应该更新为较高的那个，防止更上级重复发钱。
    lastRate = Math.max(lastRate, currentRate);
    
    // 找上级
    currentAgent = allAgents.find(a => a.id === currentAgent.parentId);
  }
  return breakdown;
};

// --- 组件 ---

const Card = ({ children, className = "" }) => (
  <div className={`bg-white border border-gray-200 rounded-xl shadow-sm p-6 ${className}`}>
    {children}
  </div>
);

const LevelBadge = ({ levelId }) => {
  const config = INITIAL_LEVELS.find(l => l.id === levelId);
  if (!config) return null;
  return (
    <span 
      className="px-2 py-0.5 rounded text-xs font-medium text-white"
      style={{ backgroundColor: config.color }}
    >
      {config.name} ({config.rate}%)
    </span>
  );
};

export default function DifferentialSystem() {
  const [activeTab, setActiveTab] = useState('dashboard');
  const [simulationAmount, setSimulationAmount] = useState(10000);
  const [selectedAgentId, setSelectedAgentId] = useState(7); // 默认选中一个底层业务员

  return (
    <div className="min-h-screen bg-gray-50 text-gray-900 font-sans flex">
      {/* 侧边栏 */}
      <aside className="w-64 bg-slate-900 text-slate-300 flex-shrink-0 flex flex-col h-screen fixed left-0 top-0 z-10">
        <div className="h-16 flex items-center px-6 border-b border-slate-800">
          <div className="flex items-center gap-2 text-blue-500 font-bold text-xl">
            <TrendingUp size={24} />
            <span className="text-white">Aurora<span className="text-blue-500">System</span></span>
          </div>
        </div>

        <nav className="flex-1 py-6 px-3 space-y-1">
          <NavButton active={activeTab === 'dashboard'} onClick={() => setActiveTab('dashboard')} icon={LayoutDashboard} label="经营概览" />
          <NavButton active={activeTab === 'team'} onClick={() => setActiveTab('team')} icon={Users} label="团队架构" />
          <NavButton active={activeTab === 'calculator'} onClick={() => setActiveTab('calculator')} icon={Calculator} label="级差试算" />
          <NavButton active={activeTab === 'commission'} onClick={() => setActiveTab('commission')} icon={Wallet} label="佣金明细" />
          <NavButton active={activeTab === 'settings'} onClick={() => setActiveTab('settings')} icon={Settings} label="制度配置" />
        </nav>
        
        <div className="p-4 border-t border-slate-800">
          <div className="flex items-center gap-3">
            <div className="w-8 h-8 rounded-full bg-blue-600 flex items-center justify-center text-white font-bold text-xs">A</div>
            <div>
              <div className="text-sm text-white font-medium">Admin User</div>
              <div className="text-xs text-slate-500">超级管理员</div>
            </div>
          </div>
        </div>
      </aside>

      {/* 主内容 */}
      <main className="flex-1 ml-64 p-8 overflow-y-auto h-screen">
        {activeTab === 'dashboard' && <DashboardView />}
        {activeTab === 'team' && <TeamTreeView agents={INITIAL_AGENTS} />}
        {activeTab === 'calculator' && (
          <CalculatorView 
            amount={simulationAmount} 
            setAmount={setSimulationAmount} 
            selectedAgentId={selectedAgentId}
            setSelectedAgentId={setSelectedAgentId}
            agents={INITIAL_AGENTS}
            levels={INITIAL_LEVELS}
          />
        )}
        {activeTab === 'commission' && <CommissionView orders={RECENT_ORDERS} />}
        {activeTab === 'settings' && <SettingsView levels={INITIAL_LEVELS} />}
      </main>
    </div>
  );
}

// --- 子视图组件 ---

function DashboardView() {
  const data = [
    { name: '周一', sales: 4000 },
    { name: '周二', sales: 3000 },
    { name: '周三', sales: 2000 },
    { name: '周四', sales: 2780 },
    { name: '周五', sales: 1890 },
    { name: '周六', sales: 2390 },
    { name: '周日', sales: 3490 },
  ];

  return (
    <div className="space-y-6">
      <header className="flex justify-between items-center mb-2">
        <div>
          <h1 className="text-2xl font-bold text-gray-900">经营概览 Dashboard</h1>
          <p className="text-gray-500 text-sm mt-1">实时监控全网业绩与拨比情况</p>
        </div>
        <button className="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-lg text-sm font-medium flex items-center gap-2 transition">
          <Plus size={16} /> 录入新业绩
        </button>
      </header>

      {/* 核心指标 */}
      <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
        <StatCard title="本月总业绩 (GMV)" value="¥2,450,000" trend="+12.5%" icon={DollarSign} color="blue" />
        <StatCard title="累计发放佣金" value="¥857,500" trend="+8.2%" icon={Wallet} color="purple" />
        <StatCard title="新增代理 (本月)" value="128 人" trend="+24.0%" icon={Users} color="emerald" />
        <StatCard title="平均拨比率" value="34.2%" trend="-1.2%" icon={TrendingUp} color="orange" desc="低于预警线 45%" />
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        {/* 业绩趋势图 */}
        <Card className="lg:col-span-2">
          <div className="flex justify-between items-center mb-6">
            <h3 className="font-bold text-gray-800">近7日业绩走势</h3>
            <select className="border border-gray-300 rounded text-sm px-2 py-1 outline-none">
              <option>本周</option>
              <option>上周</option>
            </select>
          </div>
          <div className="h-72">
            <ResponsiveContainer width="100%" height="100%">
              <BarChart data={data}>
                <CartesianGrid strokeDasharray="3 3" vertical={false} />
                <XAxis dataKey="name" axisLine={false} tickLine={false} />
                <YAxis axisLine={false} tickLine={false} />
                <Tooltip cursor={{ fill: '#f1f5f9' }} />
                <Bar dataKey="sales" fill="#3b82f6" radius={[4, 4, 0, 0]} barSize={40} />
              </BarChart>
            </ResponsiveContainer>
          </div>
        </Card>

        {/* 级别分布 */}
        <Card>
          <h3 className="font-bold text-gray-800 mb-4">代理职级分布</h3>
          <div className="h-64 relative">
             <ResponsiveContainer width="100%" height="100%">
              <PieChart>
                <Pie
                  data={[
                    { name: '董事', value: 5 },
                    { name: '合伙人', value: 15 },
                    { name: '总监', value: 45 },
                    { name: '经理', value: 120 },
                    { name: '实习', value: 300 },
                  ]}
                  innerRadius={60}
                  outerRadius={80}
                  paddingAngle={5}
                  dataKey="value"
                >
                  {INITIAL_LEVELS.map((entry, index) => (
                    <Cell key={`cell-${index}`} fill={entry.color} />
                  ))}
                </Pie>
                <Tooltip />
              </PieChart>
            </ResponsiveContainer>
            <div className="absolute inset-0 flex flex-col items-center justify-center pointer-events-none">
              <span className="text-2xl font-bold text-gray-800">485</span>
              <span className="text-xs text-gray-500">总人数</span>
            </div>
          </div>
          <div className="space-y-2 mt-4">
            {INITIAL_LEVELS.slice(0,3).map(l => (
               <div key={l.id} className="flex justify-between text-sm">
                 <span className="flex items-center gap-2">
                   <span className="w-2 h-2 rounded-full" style={{backgroundColor: l.color}}></span>
                   {l.name}
                 </span>
                 <span className="text-gray-500">{Math.floor(Math.random() * 100)}人</span>
               </div>
            ))}
          </div>
        </Card>
      </div>
    </div>
  );
}

function TeamTreeView({ agents }) {
  // 构建树形结构
  const buildTree = (parentId) => {
    return agents
      .filter(agent => agent.parentId === parentId)
      .map(agent => ({
        ...agent,
        children: buildTree(agent.id)
      }));
  };

  const treeData = buildTree(null);

  const AgentNode = ({ node, depth = 0 }) => {
    const [expanded, setExpanded] = useState(true);
    const hasChildren = node.children && node.children.length > 0;

    return (
      <div className="select-none">
        <div 
          className={`flex items-center gap-3 p-3 border-b border-gray-100 hover:bg-gray-50 transition cursor-pointer ${depth === 0 ? 'bg-gray-50' : ''}`}
          style={{ paddingLeft: `${depth * 24 + 12}px` }}
          onClick={() => setExpanded(!expanded)}
        >
          <div className="w-5 flex justify-center text-gray-400">
            {hasChildren && (expanded ? <ChevronDown size={16} /> : <ChevronRight size={16} />)}
          </div>
          
          <div className="w-8 h-8 rounded-full bg-blue-100 text-blue-600 flex items-center justify-center text-xs font-bold">
            {node.name[0]}
          </div>

          <div className="flex-1">
            <div className="flex items-center gap-2">
              <span className="font-medium text-gray-900">{node.name}</span>
              <LevelBadge levelId={node.level} />
            </div>
            <div className="text-xs text-gray-500 flex gap-4 mt-0.5">
              <span>ID: {node.id}</span>
              <span>团队: {node.teamSize}人</span>
            </div>
          </div>

          <div className="text-right pr-4">
            <div className="font-medium text-gray-900">{formatCurrency(node.sales)}</div>
            <div className="text-xs text-gray-500">累计业绩</div>
          </div>
        </div>

        {expanded && hasChildren && (
          <div>
            {node.children.map(child => (
              <AgentNode key={child.id} node={child} depth={depth + 1} />
            ))}
          </div>
        )}
      </div>
    );
  };

  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center">
        <h2 className="text-2xl font-bold">团队架构树 (Genealogy)</h2>
        <div className="relative">
          <input 
            type="text" 
            placeholder="搜索代理姓名或ID..." 
            className="pl-9 pr-4 py-2 border border-gray-300 rounded-lg text-sm focus:ring-2 focus:ring-blue-500 outline-none w-64"
          />
          <Search className="absolute left-3 top-2.5 text-gray-400" size={16} />
        </div>
      </div>

      <Card className="p-0 overflow-hidden min-h-[500px]">
        <div className="bg-gray-50 p-3 border-b border-gray-200 text-xs font-bold text-gray-500 uppercase flex">
          <div className="w-12"></div> {/* Spacer for icon/chevron */}
          <div className="flex-1 pl-2">代理信息</div>
          <div className="w-32 text-right pr-4">业绩</div>
        </div>
        {treeData.map(rootNode => (
          <AgentNode key={rootNode.id} node={rootNode} />
        ))}
      </Card>
    </div>
  );
}

function CalculatorView({ amount, setAmount, selectedAgentId, setSelectedAgentId, agents, levels }) {
  // 实时计算
  const results = calculateCommissions(amount, selectedAgentId, agents, levels);
  const totalPayout = results.reduce((sum, item) => sum + item.amount, 0);
  const selectedAgentName = agents.find(a => a.id === selectedAgentId)?.name || '未知';

  return (
    <div className="max-w-5xl mx-auto space-y-8">
      <div className="text-center">
        <h2 className="text-2xl font-bold text-gray-900">级差佣金试算引擎</h2>
        <p className="text-gray-500">模拟任意代理产生业绩，系统自动计算整条线的收益分配。</p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {/* 输入面板 */}
        <Card className="md:col-span-1 space-y-6 bg-white border-t-4 border-t-blue-500">
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-2">1. 选择出单代理</label>
            <select 
              className="w-full border border-gray-300 rounded-lg p-2.5 bg-gray-50 outline-none focus:ring-2 focus:ring-blue-500"
              value={selectedAgentId}
              onChange={(e) => setSelectedAgentId(Number(e.target.value))}
            >
              {agents.map(a => (
                <option key={a.id} value={a.id}>
                  {a.name} - {levels.find(l=>l.id===a.level)?.name}
                </option>
              ))}
            </select>
          </div>

          <div>
            <label className="block text-sm font-medium text-gray-700 mb-2">2. 输入订单金额</label>
            <div className="relative">
              <span className="absolute left-3 top-2.5 text-gray-500 font-bold">¥</span>
              <input 
                type="number" 
                value={amount}
                onChange={(e) => setAmount(Number(e.target.value))}
                className="w-full pl-8 pr-4 py-2.5 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 outline-none font-mono font-bold text-lg"
              />
            </div>
          </div>

          <div className="pt-4 border-t border-gray-100">
            <div className="flex justify-between text-sm mb-2">
              <span className="text-gray-500">总拨出金额</span>
              <span className="font-bold text-red-600">{formatCurrency(totalPayout)}</span>
            </div>
            <div className="flex justify-between text-sm">
              <span className="text-gray-500">总拨出比 (Payout)</span>
              <span className="font-bold text-gray-900">{((totalPayout/amount)*100).toFixed(2)}%</span>
            </div>
          </div>
        </Card>

        {/* 结果面板 */}
        <Card className="md:col-span-2 relative overflow-hidden">
          <div className="absolute top-0 right-0 p-4 opacity-10">
            <Award size={120} />
          </div>
          
          <h3 className="font-bold text-lg mb-4 flex items-center gap-2">
            <FileText size={20} className="text-blue-500"/>
            分润明细单据
          </h3>

          <div className="bg-gray-50 rounded-lg border border-gray-200 overflow-hidden">
            <table className="w-full text-sm text-left">
              <thead className="bg-gray-100 text-gray-500 font-medium">
                <tr>
                  <th className="p-3">受益人 (层级)</th>
                  <th className="p-3">当前职级</th>
                  <th className="p-3">级差算法</th>
                  <th className="p-3 text-right">分润金额</th>
                </tr>
              </thead>
              <tbody className="divide-y divide-gray-200">
                {results.length > 0 ? (
                  results.map((item, idx) => (
                    <tr key={idx} className="hover:bg-blue-50/50 transition">
                      <td className="p-3 font-medium text-gray-900">
                        {item.agent}
                        {idx === 0 && <span className="ml-2 text-xs bg-blue-100 text-blue-600 px-1.5 py-0.5 rounded">出单人</span>}
                      </td>
                      <td className="p-3">
                        <span className="text-xs bg-gray-200 px-2 py-0.5 rounded text-gray-700">{item.level} ({item.roleRate})</span>
                      </td>
                      <td className="p-3 text-gray-500 font-mono text-xs">
                        {amount} × <span className="text-blue-600 font-bold">{item.diffRate}</span> ({item.calculation})
                      </td>
                      <td className="p-3 text-right font-bold text-emerald-600">
                        +{formatCurrency(item.amount)}
                      </td>
                    </tr>
                  ))
                ) : (
                  <tr>
                    <td colSpan={4} className="p-8 text-center text-gray-400">
                      无分润产生 (可能是金额为0或无上级)
                    </td>
                  </tr>
                )}
              </tbody>
            </table>
          </div>

          <div className="mt-4 p-3 bg-yellow-50 text-yellow-800 text-xs rounded border border-yellow-100">
            <strong>逻辑说明：</strong> 本系统采用标准级差制。上级拿（自身比例 - 直属下线比例）的差额。如果出现平级（比例相同），则默认不再产生级差收益（平级奖需另外配置）。
          </div>
        </Card>
      </div>
    </div>
  );
}

function CommissionView({ orders }) {
  return (
    <div className="space-y-6">
      <h2 className="text-2xl font-bold">佣金流水明细</h2>
      <Card className="p-0 overflow-hidden">
        <table className="w-full text-left text-sm">
          <thead className="bg-gray-50 text-gray-500 border-b border-gray-200">
            <tr>
              <th className="p-4">订单号</th>
              <th className="p-4">出单时间</th>
              <th className="p-4">出单人</th>
              <th className="p-4 text-right">订单金额</th>
              <th className="p-4 text-center">状态</th>
              <th className="p-4 text-right">操作</th>
            </tr>
          </thead>
          <tbody className="divide-y divide-gray-100">
            {orders.map(order => (
              <tr key={order.id} className="hover:bg-gray-50">
                <td className="p-4 font-mono text-gray-600">{order.id}</td>
                <td className="p-4">{order.date}</td>
                <td className="p-4 font-medium">{order.agentName}</td>
                <td className="p-4 text-right font-medium">{formatCurrency(order.amount)}</td>
                <td className="p-4 text-center">
                  <span className="px-2 py-1 bg-green-100 text-green-700 rounded-full text-xs font-bold">已结算</span>
                </td>
                <td className="p-4 text-right">
                  <button className="text-blue-600 hover:text-blue-800 hover:underline">查看分润</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </Card>
    </div>
  );
}

function SettingsView({ levels }) {
  return (
    <div className="max-w-4xl space-y-6">
      <h2 className="text-2xl font-bold">级差制度配置</h2>
      <p className="text-gray-500">在此处定义各个职级的提成比例。修改后将影响后续所有订单的计算。</p>
      
      <Card>
        <div className="flex justify-between items-center mb-6">
          <h3 className="font-bold text-lg">职级与费率 (Levels & Rates)</h3>
          <button className="text-sm bg-blue-50 text-blue-600 px-3 py-1.5 rounded hover:bg-blue-100 transition">
            + 新增职级
          </button>
        </div>

        <div className="space-y-4">
          {levels.map((level, idx) => (
            <div key={level.id} className="flex items-center gap-4 p-4 border border-gray-100 rounded-lg hover:border-blue-200 transition bg-white">
              <div className="w-8 h-8 rounded-full flex items-center justify-center text-white font-bold text-sm" style={{backgroundColor: level.color}}>
                {level.id}
              </div>
              
              <div className="flex-1 grid grid-cols-3 gap-4">
                <div>
                  <label className="text-xs text-gray-400 block mb-1">职级名称</label>
                  <input type="text" value={level.name} className="w-full text-sm font-medium border-b border-gray-300 focus:border-blue-500 outline-none pb-1" readOnly />
                </div>
                <div>
                  <label className="text-xs text-gray-400 block mb-1">提成比例 (%)</label>
                  <input type="number" value={level.rate} className="w-full text-sm font-bold border-b border-gray-300 focus:border-blue-500 outline-none pb-1 text-blue-600" readOnly />
                </div>
                <div>
                  <label className="text-xs text-gray-400 block mb-1">考核要求 (业绩)</label>
                  <div className="text-sm text-gray-500">≥ {formatCurrency((idx + 1) * 50000)}</div>
                </div>
              </div>

              <button className="text-gray-400 hover:text-gray-600 p-2">
                <Settings size={18} />
              </button>
            </div>
          ))}
        </div>
        
        <div className="mt-6 p-4 bg-gray-50 text-sm text-gray-600 rounded">
          <p><strong>注意：</strong> 级差制的精髓在于“级差”。最高等级的比例决定了公司的最大拨出成本（Total Payout）。建议最高等级不超过 50-60%，以预留利润空间。</p>
        </div>
      </Card>
    </div>
  );
}

// 辅助组件：仪表盘统计卡片
const StatCard = ({ title, value, trend, icon: Icon, color, desc }) => {
  const colors = {
    blue: 'bg-blue-50 text-blue-600',
    purple: 'bg-purple-50 text-purple-600',
    emerald: 'bg-emerald-50 text-emerald-600',
    orange: 'bg-orange-50 text-orange-600',
  };
  
  return (
    <Card className="flex flex-col justify-between">
      <div className="flex justify-between items-start mb-4">
        <div className={`p-3 rounded-lg ${colors[color]}`}>
          <Icon size={24} />
        </div>
        <span className={`text-xs font-medium px-2 py-1 rounded ${trend.startsWith('+') ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-700'}`}>
          {trend}
        </span>
      </div>
      <div>
        <h4 className="text-gray-500 text-sm font-medium">{title}</h4>
        <div className="text-2xl font-bold text-gray-900 mt-1">{value}</div>
        {desc && <div className="text-xs text-orange-500 mt-1">{desc}</div>}
      </div>
    </Card>
  );
};

// 修复遗漏的 NavButton 组件
const NavButton = ({ active, onClick, icon: Icon, label }) => (
  <button 
    onClick={onClick} 
    className={`w-full flex items-center gap-3 px-3 py-2.5 rounded-lg text-sm font-medium transition-all ${
      active 
        ? 'bg-blue-600 text-white shadow-lg shadow-blue-900/20' 
        : 'text-slate-400 hover:text-slate-100 hover:bg-slate-900'
    }`}
  >
    <Icon size={20} />
    <span className="text-sm font-medium">{label}</span>
  </button>
);
