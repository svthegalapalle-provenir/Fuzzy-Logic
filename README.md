Code Explaination:

Match extracted signatory details (from payslips/salary certificates) to a master list based on:

•	Company Name
•	Signatory Name
•	Designation


step-wise execution

🔹 Step 1: Clean the Input

  All strings are:
  •	Lowercased
  •	Stripped of extra spaces
  •	Converted to simple format for fair comparison
    example : clean_text("ABC Corp Ltd") → "abc corp ltd"

🔹 Step 2: Fuzzy Company Name Matching
    Handle variations like "ABC Corp" ≈ "ABC Corporation"
    Logic:
  • Use TfidfVectorizer on character n-grams (3–4 characters)
  • Compute cosine similarity between extracted_company and all master_company_list
  • Select the company with highest similarity score above a threshold (default: 0.7)
    cosine_sim(company_extracted, company_master_i) > 0.7
  • If a company match is found, restrict further signatory matching to this company only.

🔹 Step 3: Score Signatory Name (score_name()): This is the most detailed logic block.

   • Tokenization:
    Split both names:
    "John Smith" → ["john", "smith"]
    "J Smith"    → ["j", "smith"]

   • Similarity Matrix:
    Compute pairwise similarity between all unmatched tokens using:
    TfidfVectorizer(analyzer="char", ngram_range=(1, 3))
    This captures character-level n-grams like:
    •	“jo”, “ohn”
    •	“sm”, “ith”
    Then compute:
    cosinesimilarity(tfidf(namepart),tfidf(refnamepart))cosine_similarity(tfidf(name_part), tfidf(ref_name_part)) 


   • Matching Logic:
     Condition	Action - Score > 0.5	Consider a valid match
     Initial match (e.g., "J" ≈ "John")	Assign partial score (0.5–0.8)
     No match	Treat as extra name (penalize)
     The matching loop continues until no unmatched tokens remain.


🔹 Step 4: Name Score Aggregation

 • Order Score:
   Evaluates sequence of matched tokens
   Scenario	Score:
    Exact sequence	1.0
    Middle names out of order	0.8
    Last/First swapped	0.6–0.7
    Random order	0.5


 • Missing Score: Penalizes missing tokens
    •	First name missing → ×0.15
    •	Last name missing → ×0.6
    •	Middle names missing
    o	100% → ×0.6
    o	≥50% → ×0.65
    o	<50% → ×0.7

 • Weighted Average Score:
    splitsscore=∑(matchscore×weight)/∑(weights)splits_score = ∑(match_score × weight) / ∑(weights) 
    Weights:
    •	First name → 1.0
    •	Middle name → 0.7
    •	Last name → 1.0

• Extra Penalty:
  If unmatched tokens remain:
  extrapenalty=0.5/len(extratokens)extra_penalty = 0.5 / len(extra_tokens) 

 • Final Name Score:
  namescore=average([orderscore,missingscore,splitsscore],weights=[0.8,1.0,1.5])namescore∗=extrapenalty×singlenamepenaltyname_score = average([order_score, missing_score, splits_score], weights=[0.8, 1.0, 1.5]) name_score *= extra_penalty × single_name_penalty 

• Special case:
   •	If all matches are initials only → score = 0.3



🔹 Step 5: Score Designation : Using score_text_similarity():

  Logic:
    •	If abbreviation is an exact match:
    •	CFO >> Chief Financial Officer → score = 0.85
    •	If abbreviation is partially matching: Compute overlap ratio:
    matchedinitials/expectedinitialsmatched_initials / expected_initials 
    Then scale:  score=0.6+0.25×overlapratioscore = 0.6 + 0.25 × overlap_ratio
    •	If no abbreviation match: Use TF-IDF + cosine similarity (same as name logic)



🔹 Step 6: Combined Score Calculation
    combinedscore=weightedaverage(namescore,designationscore,weights=[0.7,0.3])combined_score = weighted_average(name_score, designation_score, weights=[0.7, 0.3]) 
    This prioritizes the name match, while factoring in designation as a secondary check.


🔹 Step 7: Output & Thresholding
    •	The best match (per extracted row) is retained
    •	If name_score >= 0.8 → considered high confidence
    •	Output file: signatory_matching_output.csv

Columns:
•	matched_name
•	matched_designation
•	name_match_score
•	designation_match_score
•	combined_score

________________________________________

This Approach handles :

Spelling variations,
Initials,
Abbreviations (CFO etc.),
Name reordering,
Extra/missing tokens,
Fuzzy company name matching.
Weighted scoring
