# TechBridge-Agentforce-AgentScript
YAML Script for Agentforce
system:
  instructions: "You are Lana, a Tier 1 Support Assistant at TechBridge Solutions. You are professional, warm, and concise. You never reveal internal case notes, agent assignments beyond the agent's name, or pricing and discount information. You always confirm with the customer before taking any action that modifies data. When a situation is complex, emotional, or requires account-level decisions, you escalate to a human agent."

  messages:
    welcome: |
      Hi, I'm Lana, TechBridge's Tier 1 Support Assistant. How can I help you today?
    error: "Sorry, it looks like something has gone wrong."

config:
  agent_label: "Lama"
  developer_name: "Lama"
  description: "Lana is a Tier 1 Support Assistant at TechBridge Solutions. She helps customers with case lookups, product troubleshooting, billing questions, and escalates complex issues to human agents."
  default_agent_user: "lama@00dfj00000gn1ol2080751880.ext"

language:
  default_locale: "en_US"
  additional_locales: "en_GB,vi"
  all_additional_locales: False

variables:
    contact_record: mutable object
        description: "Stores the contact record retrieved by email lookup."
        visibility: "Internal"
    case_record: mutable object
        description: "Stores the case record created or retrieved for the customer."
        visibility: "Internal"
    account_record: mutable object
        description: "Stores the customer's account record retrieved by email lookup."
        visibility: "Internal"
    case_list: mutable list[object]
        description: "Stores the list of cases returned by contact email lookup."
        visibility: "Internal"
    ContactId: linked string
        source: @MessagingEndUser.ContactId
        description: "This variable may also be referred to as MessagingEndUser ContactId"
        visibility: "External"
    contactId: mutable string                          # ← ADD THIS
        description: "Stores the Contact Id retrieved by email lookup."
        visibility: "Internal"
    caseId: mutable string                    # ← ADD THIS
        description: "Stores the Salesforce Case Id for the active case in the conversation."
        visibility: "Internal"
    
 

knowledge:
    rag_feature_config_id: "ARFPC_1JDfj000007Hbf3GAC"
    citations_enabled: True
    citations_url: "https://orgfarm-dd40c34330-dev-ed.develop.my.site.com"

start_agent topic_selector:
    label: "Topic Selector"

    description: "Determine the appropriate topic based on user input"

    reasoning:
        instructions: ->
            | Select the best topic based on the customer's intent.
              If the customer is providing data such as an email address, case number,
              or other details in response to a previous question, route them to the
              same topic that was previously active based on the context of the
              conversation history. Do not treat data responses as new topic requests.

        actions:
            go_to_Check_My_Case_Status: @utils.transition to @topic.Check_My_Case_Status
                description: "Route to case status lookup. Use this when the customer wants to check their case status, OR when the customer is providing a case number, email address, or account details in response to a previous question about their case. If the conversation history shows the agent previously asked for case details, always route here when the customer responds with that data."

            go_to_Knowledge_Troubleshooting: @utils.transition to @topic.Knowledge_Troubleshooting
                description: "Route to knowledge base troubleshooting"

            go_to_Billing_and_Account_Questions: @utils.transition to @topic.Billing_and_Account_Questions
                description: "Route to billing and account questions"

            go_to_Human_Escalation_Handoff: @utils.transition to @topic.Human_Escalation_Handoff
                description: "Route to human escalation handoff"

topic Check_My_Case_Status:
    label: "Check My Case Status"

    description: "Assist customers in checking the status of their support cases. Handles lookup by case number or email address. Also handles when customers provide case numbers, email addresses, or other identifying information in response to questions about their case. Offers to close cases that are resolved."

    reasoning:
        instructions: ->
            | IMPORTANT: Never acknowledge or thank the customer for providing information.
              Always execute the required action immediately upon receiving needed data.
  
              When the customer asks to check case status:
              - Ask for BOTH their case number AND email address in a single message.
              - Do NOT proceed until you have both pieces of information.
              - Once you have BOTH, IMMEDIATELY EXECUTE QueryAccountByEmailV3 with the email.
              - Do NOT say anything between actions.
              - IMMEDIATELY AFTER QueryAccountByEmailV3 completes, EXECUTE
            | TechBridgeGetCaseByCaseNumberV2 with {!@variables.contact_record}  and the case number.
            | - Only after BOTH actions complete, present:
              Case Number, Status, Last Updated Date, Assigned Agent Name.
  
              When the customer provides only an email address with no case number:
              - IMMEDIATELY EXECUTE QueryAccountByEmailV3 with the email.
            | - IMMEDIATELY AFTER, EXECUTE TechBridgeGetCasesforContact with {!@variables.contact_record} .
            | - List the cases returned and ask which one they want details on.
              - EXECUTE TechBridgeGetCaseByCaseNumberV2 for the selected case.
              - Present the results.
  
              If the case Status is Resolved:
              - Ask: "Has your issue been fully resolved?"
              - If yes, EXECUTE CloseCase with caseStatus = "Closed".
              - Never close a case without explicit customer confirmation.
  
              If the customer becomes frustrated or asks for a human, transition to
              Human Escalation Handoff topic immediately.

        actions:
            QueryAccountByEmailV3: @actions.QueryAccountByEmailV3
                with contact_email = ...
                set @variables.account_record = @outputs.accountRecord
                set @variables.contact_record = @outputs.contactRecord
                set @variables.contactId = @outputs.contactId

            TechBridgeGetCaseByCaseNumberV2: @actions.TechBridgeGetCaseByCaseNumberV2
                with contactRecord = @variables.contact_record
                with caseNumber = ...
                set @variables.case_record = @outputs.caseRecord
                set @variables.caseId = @outputs.caseId

            TechBridgeGetCasesforContact: @actions.TechBridgeGetCasesforContact
                with contactRecord = @variables.contact_record
                set @variables.case_list = @outputs.caseListSummary

            CloseCase: @actions.CloseCase
                with caseRecord = @variables.case_record
                with caseStatus = "Closed"
                with caseReason = ...




    actions:
        QueryAccountByEmailV3:
            description: "Look for account records by contact email"
            label: "QueryAccountByEmail"
            require_user_confirmation: False
            include_in_progress_indicator: True
            progress_indicator_message: "Retrieving data"
            source: "QueryAccountByEmailV3"
            target: "flow://QueryAccountByEmailV3"
                                                                                                                        
            inputs:
                "contact_email": string
                    description: "Stores the contact email of the asking customer"
                    label: "contact_email"
                    is_required: True
                    is_user_input: False
                                                                                                                        
            outputs:
                "accountRecord": object
                    description: "stores the account record found by contact email."
                    label: "accountRecord"
                    is_displayable: False
                    filter_from_agent: False
                    complex_data_type_name: "lightning__recordInfoType"
                                
                "account_record": object
                    description: "stores the account record found by contact email."
                    label: "account_record"
                    is_displayable: False
                    filter_from_agent: False
                    complex_data_type_name: "lightning__recordInfoType"
                                
                "contactRecord": object
                    description: "stores contact records associated with the contact email"
                    label: "contactRecord"
                    is_displayable: True
                    filter_from_agent: False
                    complex_data_type_name: "lightning__recordInfoType"
                                
                "accountNumber": string
                    description: "The customer's account number."
                    label: "Account Number"
                    is_displayable: True
                    filter_from_agent: False
                
                "contactId": string
                    description: "The Salesforce Contact Id of the customer."
                    label: "Contact Id"
                    is_displayable: False
                    filter_from_agent: False

        TechBridgeGetCaseByCaseNumberV2:
          description: "Returns a case associated with a given contact record and case number."
          inputs:
            contactRecord: object
              description: "Stores the contact record of the customer to be used with the case number to look up a case."
              label: "Contact record"
              is_required: True
              is_user_input: False
              complex_data_type_name: "lightning__recordInfoType"
            caseNumber: string
              description: "Stores the case number provided by the customer."
              label: "Case Number"
              is_required: True
              is_user_input: False
          outputs:
            caseRecord: object
              description: "Stores the case record based on the contact record and case number."
              label: "Case record"
              complex_data_type_name: "lightning__recordInfoType"
              filter_from_agent: False
              is_displayable: True
            caseStatus: string
              description: "The current status of the case."
              label: "Case Status"
              filter_from_agent: False
              is_displayable: True
            caseLastModifiedDate: object
              description: "The date the case was last updated."
              label: "Last Updated Date"
              complex_data_type_name: "lightning__dateType"
              filter_from_agent: False
              is_displayable: True
            caseOwnerId: string
              description: "The name of the agent assigned to the case."
              label: "Assigned Agent Name"
              filter_from_agent: False
              is_displayable: True
            "caseId": string
              description: "The Salesforce Id of the created At Risk case."
              label: "Case Id"
              is_displayable: False
              filter_from_agent: False      
                
                
          target: "flow://TechBridgeGetCaseByCaseNumberV2"
          label: "TechBridgeGetCaseByCaseNumber"
          require_user_confirmation: False
          include_in_progress_indicator: True
          source: "TechBridgeGetCaseByCaseNumberV2"

        TechBridgeGetCasesforContact:
            description: "Returns a list of cases related to a given Contact record."
            label: "TechBridge Get Cases For Contact"
            require_user_confirmation: False
            include_in_progress_indicator: True
            source: "TechBridgeGetCasesforContact"
            target: "flow://TechBridgeGetCasesforContact"
                                                                                                        
            inputs:
                "contactRecord": object
                    description: "The contact record for a customer used to look up related case records."
                    label: "Contact record"
                    is_required: True
                    is_user_input: False
                    complex_data_type_name: "lightning__recordInfoType"
                                                                                                        
            outputs:
                "caseList": list[object]
                    description: "Raw collection of case records - not used by agent directly."
                    label: "Case List"
                    is_displayable: False
                    filter_from_agent: True
                    complex_data_type_name: "lightning__recordInfoType"
                "caseListSummary": list[object]
                    description: "A readable summary of the customer cases including case number, subject, and status."
                    label: "Case List Summary"
                    is_displayable: True
                    filter_from_agent: False
                    complex_data_type_name: "lightning__textType"

        CloseCase:
            description: "From the case ID or the Close Case quick action, sets the case status to Closed as long as the previous status was set to Resolved, Waiting for Customer Confirmation, or a custom status."
            label: "Close Case"
            require_user_confirmation: False
            include_in_progress_indicator: True
            source: "SvcCopilotTmpl__CloseCase"
            target: "flow://SvcCopilotTmpl__CloseCase"
                                                                                                        
            inputs:
                "caseRecord": object
                    description: "Stores the case record of the case to close."
                    label: "Case record"
                    is_required: True
                    is_user_input: False
                    complex_data_type_name: "lightning__recordInfoType"
                "caseStatus": string
                    description: "Stores the case status picklist value provided by the user."
                    label: "Case Status"
                    is_required: True
                    is_user_input: True
                "caseReason": string
                    description: "Stores the case reason picklist value provided by the user."
                    label: "Case Reason"
                    is_required: False
                    is_user_input: True
                                                                                                        
            outputs:
                "SuccessMessage": string
                    description: "Stores the success message if the case was closed."
                    label: "Success Message"
                    is_displayable: False
                    filter_from_agent: False
                "ErrorMessage": string
                    description: "Stores the error message if the case wasn't closed."
                    label: "Error Message"
                    is_displayable: False
                    filter_from_agent: False

topic Knowledge_Troubleshooting:
    label: "Knowledge Troubleshooting"

    description: "Assist customers in resolving technical issues by searching the Knowledge Base and providing step-by-step guidance. Creates a support case if the issue cannot be self-resolved."

    reasoning:
        instructions: ->
            | Listen to the customer's description of their technical issue.
              Use the search_knowledge_articles action to search the Knowledge Base.
              Present the top 2 relevant articles as plain-language step-by-step instructions. Ì there is only 1 relevant article, display that 1 article as plain-language step-by-step instructions.
              Do not present them as links — summarize the steps directly in conversation.
              Ask: "Did this resolve your issue?"
              If YES:
              - Thank the customer.
              - End the session. Do not create a case.
              If NO after two article attempts:
              - Tell the customer you will create a support case for them.
              - Collect a clear description of the issue.
              - Confirm with the customer before creating the case.
              - Use the create_case action.
              - Case Origin and Type are set automatically — do not ask the customer for these.
              If the customer asks to speak to a human at any point:
              - Transition to the escalation topic immediately.
              If the customer becomes frustrated, asks repeatedly for a human, or the issue
              cannot be resolved through case lookup alone, transition to the
              Human Escalation Handoff topic.

        actions:
            AnswerQuestionsWithKnowledge: @actions.AnswerQuestionsWithKnowledge
                with query = ...
                with ragFeatureConfigId = "ARFPC_1JDfj000007Hbf3GAC"
                with citationsEnabled = True
                with citationsUrl = "https://orgfarm-dd40c34330-dev-ed.develop.my.site.com"

            TechBridgeCaseCreationAI: @actions.TechBridgeCaseCreationAI
                with caseDescription = ...
                with caseSubject = ...
                with contactRecord = ...






    actions:
        AnswerQuestionsWithKnowledge:
            description: "Searches knowledge articles to answer questions about policies, troubleshooting steps, and product information."
            require_user_confirmation: False
            include_in_progress_indicator: True
            progress_indicator_message: "Getting answers"
            source: "EmployeeCopilot__AnswerQuestionsWithKnowledge"
            target: "standardInvocableAction://streamKnowledgeSearch"
                                                                                                                                                                                
            inputs:
                "query": string
                    description: "Required. A string created by generative AI to be used in the knowledge article search."
                    label: "Query"
                    is_required: True
                    is_user_input: True
                "citationsUrl": string
                    description: "The URL to use for citations for custom Agents."
                    label: "Citations Url"
                    is_required: False
                    is_user_input: False
                "ragFeatureConfigId": string
                    description: "The RAG Feature ID to use for grounding this copilot action invocation."
                    label: "RAG Feature Configuration Id"
                    is_required: False
                    is_user_input: False
                "citationsEnabled": boolean
                    description: "Whether or not citations are enabled."
                    label: "Citations Enabled"
                    is_required: False
                    is_user_input: False
                                                                                                                           
            outputs:
                "knowledgeSummary": object
                    description: "A string formatted as rich text that includes a summary of the information retrieved from the knowledge articles and citations to those articles."
                    label: "Knowledge Summary"
                    is_displayable: True
                    filter_from_agent: False
                    complex_data_type_name: "lightning__richTextType"
                "citationSources": object
                    description: "Source links for the chunks in the hydrated prompt that's used by the planner service."
                    label: "Citation Sources"
                    is_displayable: False
                    filter_from_agent: False
                    complex_data_type_name: "@apexClassType/AiCopilot__GenAiCitationInput"

        TechBridgeCaseCreationAI:
            description: "Let a customer create a case."
            label: "TechBridgeCaseCreationAI"
            require_user_confirmation: True
            include_in_progress_indicator: True
            source: "TechBridgeCaseCreationAI"
            target: "flow://TechBridgeCaseCreationAI"
                        
            inputs:
                caseDescription: string
                    description: "Stores the details of the user issue."
                    label: "caseDescription"
                    is_required: True
                    is_user_input: False
                caseSubject: string
                    description: "Stores the subject of the case."
                    label: "caseSubject"
                    is_required: True
                    is_user_input: True
                contactRecord: object
                    description: "Stores the contact record for the case."
                    label: "contactRecord"
                    is_required: True
                    is_user_input: False
                    complex_data_type_name: "lightning__recordInfoType"
                        
            outputs:
                caseRecord: object
                    description: "Stores the case record created."
                    label: "caseRecord"
                    complex_data_type_name: "lightning__recordInfoType"
                    filter_from_agent: False
                    is_displayable: True

topic Billing_and_Account_Questions:
    label: "Billing and Account Questions"

    description: "Route to billing and account questions, including subscription inquiries, contract details, cancellation requests, and at-risk account handling. Always route here first when a customer mentions cancellation or dissatisfaction — do NOT route directly to Human Escalation for these."

    reasoning:
        instructions: ->
            | Ask for the customer's email address to look up their account.
              Once have it, IMMEDIATELY EXECUTE QueryAccountByEmailV3 action with the retrieve email to retrieve their Account record.
              IMMEDIATELY AFTER EXECUTE QueryActiveContractV2 action to retrieve their active Contract.
              Share with the customer:
              - Subscription tier (from Contract)
              - Start date (from Contract)
              - Contract Terms (from Contract)
              - Contact Status (from Contact)
              - Account Number (from Account)
              Do NOT share:
              - Pricing details
              - Discount amounts
              - Redirect these to their Account Executive
              If the customer expresses dissatisfaction or mentions cancellation:
              - Do not argue or negotiate.
              - ALWAYS EXECUTE CreateAtRiskCaseV2 immediately.
              - Tell the customer: "I'm flagging this for your dedicated success manager
              who will reach out within 2 business hours."
              If the customer becomes frustrated, asks repeatedly for a human, or the issue
              cannot be resolved through case lookup alone, transition to the
              Human Escalation Handoff topic.

        actions:
            QueryAccountByEmailV3: @actions.QueryAccountByEmailV3
                with contact_email = ...
                set @variables.account_record = @outputs.accountRecord
                set @variables.contact_record = @outputs.contactRecord
                set @variables.contactId = @outputs.contactId

            QueryActiveContractV2: @actions.QueryActiveContractV2
                with account_record = @variables.account_record

            CreateAtRiskCaseV2: @actions.CreateAtRiskCaseV2
                with ContactId = @variables.contactId
                with caseDescription = "Customer expressed dissatisfaction or mentioned cancellation."
                with caseSubject = "High Priority At-Risk Case"
                set @variables.caseId = @outputs.caseId



    actions:
        QueryAccountByEmailV3:
            description: "Look for account records by contact email"
            label: "QueryAccountByEmail"
            require_user_confirmation: False
            include_in_progress_indicator: True
            progress_indicator_message: "Retrieving data"
            source: "QueryAccountByEmailV3"
            target: "flow://QueryAccountByEmailV3"
                                                                                                                
            inputs:
                "contact_email": string
                    description: "Stores the contact email of the asking customer"
                    label: "contact_email"
                    is_required: True
                    is_user_input: False
                                                                                                                
            outputs:
                "accountRecord": object
                    description: "stores the account record found by contact email."
                    label: "accountRecord"
                    is_displayable: False
                    filter_from_agent: False
                    complex_data_type_name: "lightning__recordInfoType"
                        
                "account_record": object
                    description: "stores the account record found by contact email."
                    label: "account_record"
                    is_displayable: False
                    filter_from_agent: False
                    complex_data_type_name: "lightning__recordInfoType"
                        
                "contactRecord": object
                    description: "stores contact records associated with the contact email"
                    label: "contactRecord"
                    is_displayable: True
                    filter_from_agent: False
                    complex_data_type_name: "lightning__recordInfoType"
                
                "accountNumber": string
                    description: "The customer's account number."
                    label: "Account Number"
                    is_displayable: True
                    filter_from_agent: False
                
                "contactId": string
                    description: "The Salesforce Contact Id of the customer."
                    label: "Contact Id"
                    is_displayable: False
                    filter_from_agent: False

        QueryActiveContractV2:
            description: "Look for active contracts associated with the account"
            label: "QueryActiveContract"
            require_user_confirmation: False
            include_in_progress_indicator: True
            progress_indicator_message: "Retrieiving data"
            source: "QueryActiveContractV2"
            target: "flow://QueryActiveContractV2"
                                                                                                                
            inputs:
                "account_record": object
                    description: "stores the account record of the customer"
                    label: "account_record"
                    is_required: True
                    is_user_input: False
                    complex_data_type_name: "lightning__recordInfoType"
                                                                                                                
            outputs:
                "contractRecord": object
                    description: "stores the contract records associated with customer's account."
                    label: "contractRecord"
                    is_displayable: True
                    filter_from_agent: False
                    complex_data_type_name: "lightning__recordInfoType"
                
                "subscriptionTier": string
                    description: "The customer's subscription tier."
                    label: "Subscription Tier"
                    is_displayable: True
                    filter_from_agent: False
                "contractStartDate": string
                    description: "The contract start date."
                    label: "Contract Start Date"
                    is_displayable: True
                    filter_from_agent: False
                "contractTerm": string
                    description: "The contract term in months."
                    label: "Contract Term"
                    is_displayable: True
                    filter_from_agent: False
                "contractStatus": string
                    description: "The current status of the contract."
                    label: "Contract Status"
                    is_displayable: True
                    filter_from_agent: False

        CreateAtRiskCaseV2:
            description: "Create an At Risk Case when escalated"
            label: "CreateAtRiskCase"
            require_user_confirmation: True
            include_in_progress_indicator: True
            progress_indicator_message: "Retrieving data"
            source: "CreateAtRiskCaseV2"
            target: "flow://CreateAtRiskCaseV2"
                                                                                                
            inputs:
                "ContactId": string
                    description: "Stores the contact id of the customer"
                    label: "ContactId"
                    is_required: True
                    is_user_input: False
                "caseDescription": string
                    description: "stores case description"
                    label: "caseDescription"
                    is_required: True
                    is_user_input: False
                "caseSubject": string
                    description: "Stores case subject"
                    label: "caseSubject"
                    is_required: False
                    is_user_input: False
                                                                                                
            outputs:
                "caseRecord": object
                    description: "Stores the created at risk case record"
                    label: "caseRecord"
                    is_displayable: True
                    filter_from_agent: False
                    complex_data_type_name: "lightning__recordInfoType"
                "caseId": string
                    description: "The Salesforce Id of the created At Risk case."
                    label: "Case Id"
                    is_displayable: False
                    filter_from_agent: False

topic Human_Escalation_Handoff:
    label: "Human Escalation Handoff"

    description: "Handles the structured handoff from Lana to a human support agent. Summarizes the conversation, updates the case and informs the customer."

    reasoning:
        instructions: ->
            | Run these steps in exact order when escalating. Do not skip steps.
              Step 1: Summarize the conversation so far into a clear 3-5 sentence handoff note
              covering: what the customer needs, what was already tried, and why escalation
              was triggered.
              Step 2: EXECUTE Escalate_to_Human_Agent using @variables.caseId and the handoff
              summary as handoff_notes. This action will update the case Status to "Escalated",
              populate the handoff notes field, set Origin to "Agentforce Troubleshooting",
              and assign the case to the support queue.
              Step 3: Tell the customer:
              "I've connected you with a member of our team who has full context of our
              conversation. They'll be with you shortly. Your case number is [case number]."
              If no case record exists yet (customer escalated without a prior case lookup),
              inform the customer that a team member will reach out and ask them to provide
              their email so an agent can locate their account.
              Do not end until all four steps are completed.


        actions:
            Escalate_to_Human_Agent: @actions.Escalate_to_Human_Agent
                with caseId = @variables.caseId
                with handoff_notes = ...
                with queueId = "00Gfj000008IwLt"



    actions:
        Escalate_to_Human_Agent:
            description: "Escalates the Case to the support queue with an AI-generated handoff summary."
            label: "Escalate to Human Agent"
            require_user_confirmation: False
            include_in_progress_indicator: True
            progress_indicator_message: "Escalating to support team"
            source: "Escalate_to_Human_Agent"
            target: "flow://Escalate_to_Human_Agent"
                        
            inputs:
                caseId: string
                    description: "Stores the ID of the case record to escalate."
                    label: "caseId"
                    is_required: True
                    is_user_input: False
                handoff_notes: string
                    description: "Stores the AI-generated handoff summary."
                    label: "handoff_notes"
                    is_required: True
                    is_user_input: False
                queueId: string
                    description: "Stores the queue ID to route the case to."
                    label: "queueId"
                    is_required: True
                    is_user_input: False
                        
            outputs:
                isSuccess: boolean
                    description: "Stores whether the escalation succeeded."
                    label: "isSuccess"
                    filter_from_agent: False
                    is_displayable: False

connection messaging:
    escalation_message: "One moment, I'm transferring our conversation to get you more help."
    outbound_route_type: "OmniChannelFlow"
    outbound_route_name: "flow://Routing_Support_Agent"
    adaptive_response_allowed: True
